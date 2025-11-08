使用 CMake 生成 C++ Xcode 工程时，可以通过 `set_target_properties` 来设置 Xcode 工程目标 **Scheme** 的 **Environment Variables** 。需要注意的是：此项设置需要 CMake 版本为 **3.25+**：

```cmake
set_target_properties(${PROJECT_NAME} PROPERTIES
    XCODE_GENERATE_SCHEME "TRUE"    # 如果设置不成功可能是因为此开关没有设置为 TRUE
    XCODE_SCHEME_ENVIRONMENT "VK_LAYER_PATH=/Users/czw/Documents/VulkanSDK/1.4.313.1/macOS/share/vulkan/explicit_layer.d;VK_ICD_FILENAMES=/Users/czw/Documents/VulkanSDK/1.4.313.1/macOS/share/vulkan/icd.d/MoltenVK_icd.json"
)
```
