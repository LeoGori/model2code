# Changelog - fbrand-new Contributions to model2code

## Executive Summary

These contributions focused on **stabilizing and improving the code generation pipeline** for the model2code project. The main areas of improvement include:

1. **Type System Improvements** - Better handling of data types from SCXML datamodels
2. **Service/Topic Parsing** - Fixed parsing of ROS service and topic names  
3. **Template Generation** - Improved skill template generation for ROS2
4. **Stability Fixes** - Prevented crashes and incorrect code output

---

## Detailed Changes

### 1. Response Field Type Handling

**Commit:** `4cb8a93`

#### Problem
Generated code was not accounting for the actual field types in service responses. All fields were treated as strings, causing compilation errors for non-string types.

#### Solution
Added interface file parsing and type-aware code generation.

**Updated code generation** in `src/Replacer.cpp`:
```cpp
// Create the appropriate field access expression based on type
std::string fieldAccess;
if (fieldType == "string") {
    fieldAccess = "response->" + fieldName + ".c_str()";  // String needs c_str()
} else {
    fieldAccess = "response->" + fieldName;  // Other types used directly
}
```

In order to get the correct type we need to somehow extract it from the files we have which leads us to:

---

### 2. Data Type Mapping via SCXML Location Attribute

**Commit:** `3b1862b`

#### Problem
The code generator was unable to correctly map response field types from service interfaces to the appropriate datamodel variables, causing type mismatches in generated code.

#### Solution
Implemented a new mapping system that extracts data types from the SCXML datamodel by using the `location` attribute in `<assign>` tags.

Let's take as an example the [IsCheckingForPeopleSkill](https://github.com/convince-project/UC3/blob/main/model-high-level/Skills/IsCheckingForPeopleSkill.scxml): 

For example, at line [31-37](https://github.com/convince-project/UC3/blob/main/model-high-level/Skills/IsCheckingForPeopleSkill.scxml#L31-L37) we have an `assign` tag inside the `ros_service_handle_response`
```
<state id="query">
<ros_service_handle_response name="/BlackboardComponent/GetInt" target="evaluate">
    <assign location="m_value"  expr="_res.value"/>
    <!-- Fix: Use is_ok instead of non-existent result field -->
    <assign location="m_result" expr="_res.is_ok"/>
</ros_service_handle_response>
</state>
```

From the same file we then lookup for the `m_value` and `m_result` field in the datamodel to extract their
types.

[Lines 9-15](https://github.com/convince-project/UC3/blob/main/model-high-level/Skills/IsCheckingForPeopleSkill.scxml#L9C1-L15C15)
```
<datamodel>
    <data id="m_name"         type="string"  expr="'isCheckingForPeople'"/>
    <data id="m_value"        type="int32"   expr="0"/>
    <data id="m_result"       type="bool"    expr="false"/>
    <data id="SKILL_SUCCESS"  type="int8"    expr="0"/>
    <data id="SKILL_FAILURE"  type="int8"    expr="1"/>
</datamodel>
```

#### Code Changes

**New data structure** in `include/Data.h`:
```cpp
// Added to eventDataStr struct (line 84)
std::map<std::string, std::string> responseFieldToDatamodelMap;
// Mapping from response fields to datamodel variables (e.g., "param" -> "m_param")
```

**New function** in `include/ExtractFromXML.h`:
```cpp
bool getInterfaceFieldsFromAssignTag(
    tinyxml2::XMLElement* element, 
    std::vector<std::string>& interfaceFields, 
    std::map<std::string, std::string>& responseFieldToDatamodelMap
);
```

**Core logic** in `src/ExtractFromXML.cpp` (lines 370-406):
```cpp
// Extract response field name from expr (e.g., "_res.param" -> "param")
std::string exprStr(expr);
size_t dotPos = exprStr.find('.');
if (dotPos != std::string::npos) {
    std::string responseField = exprStr.substr(dotPos + 1);
    std::string datamodelVar(location);
    
    // Store the mapping from response field to datamodel variable
    responseFieldToDatamodelMap[responseField] = datamodelVar;
    interfaceFields.push_back(responseField);
}
```

**Updated replacer logic** in `src/Replacer.cpp` (lines 262-280):
```cpp
// Look up the datamodel variable for this response field
std::string datamodelVar;
auto mappingIt = eventData.responseFieldToDatamodelMap.find(responseField);
if (mappingIt != eventData.responseFieldToDatamodelMap.end()) {
    datamodelVar = mappingIt->second;
} else {
    // Fallback to the old logic if mapping is not found
    datamodelVar = "m_" + fieldName;
}

// Get the type from interfaceData using the datamodel variable name
auto pos = eventData.interfaceData.find(datamodelVar);
```

---

### 3. Service Client Initialization in Skill Start

**Commits:** `f0ad936`, `d24bc2b` | **Dates:** 2025-11-24, 2025-11-25 | **Type:** ARCHITECTURAL FIX

#### Problem
Service clients were being created inside the event callback lambda, causing repeated client creation on every event hence slowing down the whole application

#### Solution
Moved service client creation to the `start()` method, with proper service availability checking at startup.

#### Code Changes

**New member declarations** in `template_skill/include/TemplateSkill.h`:
```cpp
/*SERVICE_CLIENTS_LIST*//*SERVICE_CLIENT*/
std::shared_ptr<rclcpp::Node> $eventData.nodeName$;
std::shared_ptr<rclcpp::Client<$eventData.interfaceName$::srv::$eventData.functionName$>> $eventData.clientName$;
/*END_SERVICE_CLIENT*/
```

**Moved to start()** in `template_skill/src/TemplateSkill.cpp`:
```cpp
// Service client creation now in start(), not in event callback
$eventData.nodeName$ = rclcpp::Node::make_shared(m_name + "SkillNode$eventData.functionName$");
$eventData.clientName$ = $eventData.nodeName$->create_client<...>($eventData.serverName$);

// Service availability check with timeout at startup
bool wait_succeded{true};
int retries = 0;
while (!$eventData.clientName$->wait_for_service(std::chrono::seconds(1))) {
    if (!rclcpp::ok() || retries++ == SERVICE_TIMEOUT) {
        wait_succeded = false;
        break;
    }
}
if (!wait_succeded) {
    std::exit(1);  // Fail fast if service unavailable
}
```

**New data field** in `include/Data.h`:
```cpp
std::string serviceClientH;  // Template for service client header declarations
```

**New replacer handling** in `src/Replacer.cpp`:
```cpp
std::string serviceClientH = savedCode.serviceClientH;
replaceCommonEventPlaceholders(serviceClientH, eventData);
writeAfterCommand(str, "/*SERVICE_CLIENTS_LIST*/", serviceClientH);
```

---

### 4. Blackboard/Service Type Name Fix

**Commit:** `e3f26a5` | **Date:** 2025-09-16 | **Type:** BUG FIX

#### Problem
When using blackboard interfaces (e.g., `blackboard_interfaces/GetIntBlackboard`), the generated code was incorrectly using the function name instead of the service type name for ROS2 service client templates.

#### Solution
Added separate `serviceTypeName` field to properly distinguish between the function being called and the interface type.

#### Code Changes

**New fields** in `include/Data.h`:
```cpp
std::string serviceTypeName;           // e.g., "GetIntBlackboard"
std::string serviceTypeNameSnakeCase;  // e.g., "get_int_blackboard"
```

**Extraction logic** in `src/ExtractFromXML.cpp`:
```cpp
size_t lastSlash = eventData.messageInterfaceType.find_last_of("/");
if (lastSlash != std::string::npos) {
    eventData.serviceTypeName = eventData.messageInterfaceType.substr(lastSlash + 1);
    turnToSnakeCase(eventData.serviceTypeName, eventData.serviceTypeNameSnakeCase);
}
```

**Template fix** in `template_skill/src/TemplateSkill.cpp`:
```cpp
// Changed from (incorrect - used function name):
std::shared_ptr<rclcpp::Client<$eventData.interfaceName$::srv::$eventData.functionName$>>

// To (correct - uses service type):
std::shared_ptr<rclcpp::Client<$eventData.interfaceName$::srv::$eventData.serviceTypeName$>>
```

---

## Commit History (Chronological)

| Date | Commit | Type | Summary |
|------|--------|------|---------|
| 2025-09-10 | `c5e885f` | fix | Wrong expected topic name in battery level skill |
| 2025-09-10 | `18302c1` | fix | Topic name parsing for multi-part names |
| 2025-09-11 | `a978d99` | fix | Correct component topic name |
| 2025-09-11 | `a205446` | fix | Topic callback generation |
| 2025-09-11 | `3ffe328` | fix | `msg_.x` → `_event.data.x` translation |
| 2025-09-11 | `4cb8a93` | fix | Field type handling in responses |
| 2025-09-11 | `ed7ee75` | fix | checkIfStartSkill test |
| 2025-09-16 | `e3f26a5` | fix | Blackboard/service type names |
| 2025-09-16 | `9905d7e` | fix | HL SCXML naming issues |
| 2025-09-16 | `2509e9a` | fix | Minor HL SCXML fixes |
| 2025-09-16 | `41505ca` | fix | **Segfault for unused services** |
| 2025-09-17 | `7c684eb` | fix | **Data types from SCXML not interface files** |
| 2025-09-30 | `37a62bd` | fix | interfaceResponseFields lookup |
| 2025-10-01 | `3b1862b` | **feat** | **Data type mapping via location attribute** |
| 2025-10-02 | `5d87317` | fix | Remove Python code leftover (930 lines) |
| 2025-11-24 | `f0ad936` | fix | **Service client init in skill start** |
| 2025-11-25 | `d24bc2b` | fix | Service client list in header |

---

## Files Most Affected

| File | Changes | Purpose |
|------|---------|---------|
| [src/Replacer.cpp](src/Replacer.cpp) | 7 commits | Code generation and template substitution |
| [src/ExtractFromXML.cpp](src/ExtractFromXML.cpp) | 6 commits | SCXML parsing and data extraction |
| [template_skill/src/TemplateSkill.cpp](template_skill/src/TemplateSkill.cpp) | 4 commits | Skill template for ROS2 generation |
| [include/ExtractFromXML.h](include/ExtractFromXML.h) | 3 commits | Parser function declarations |
| [include/Data.h](include/Data.h) | 3 commits | Data structures for extracted information |

