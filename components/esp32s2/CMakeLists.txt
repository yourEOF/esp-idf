idf_build_get_property(target IDF_TARGET)
idf_build_get_property(sdkconfig_header SDKCONFIG_HEADER)
if(NOT "${target}" STREQUAL "esp32s2")
    return()
endif()

if(BOOTLOADER_BUILD)
    # For bootloader, all we need from esp32s2 is headers
    idf_component_register(INCLUDE_DIRS include)
    target_linker_script(${COMPONENT_LIB} INTERFACE "ld/esp32s2.peripherals.ld")
else()
    # Regular app build

    set(srcs "cache_err_int.c"
             "memprot.c"
             "clk.c"
             "crosscore_int.c"
             "dport_access.c"
             "hw_random.c"
             "intr_alloc.c"
             "pm_esp32s2.c"
             "pm_trace.c"
             "sleep_modes.c"
             "spiram.c"
             "spiram_psram.c"
             "system_api_esp32s2.c"
             "esp_crypto_lock.c"
             "esp_hmac.c"
             "esp_ds.c")

    set(include_dirs "include")

    set(requires driver efuse)

    # driver is a public requirement because esp_sleep.h uses gpio_num_t & touch_pad_t
    # app_update is added here because cpu_start.c uses esp_ota_get_app_description() function.
    # esp_timer is added here because cpu_start.c uses esp_timer
    set(priv_requires
        app_trace app_update bootloader_support log mbedtls nvs_flash
        pthread spi_flash vfs espcoredump esp_common esp_timer)

    set(fragments linker.lf ld/esp32s2_fragments.lf)

    idf_component_register(SRCS "${srcs}"
                        INCLUDE_DIRS "${include_dirs}"
                        LDFRAGMENTS "${fragments}"
                        REQUIRES "${requires}"
                        PRIV_REQUIRES "${priv_requires}"
                        REQUIRED_IDF_TARGETS esp32s2)

    target_linker_script(${COMPONENT_LIB} INTERFACE "${CMAKE_CURRENT_BINARY_DIR}/esp32s2_out.ld")

    # Process the template file through the linker script generation mechanism, and use the output for linking the
    # final binary
    target_linker_script(${COMPONENT_LIB} INTERFACE "${CMAKE_CURRENT_LIST_DIR}/ld/esp32s2.project.ld.in" PROCESS "${CMAKE_CURRENT_BINARY_DIR}/ld/esp32s2.project.ld")

    target_linker_script(${COMPONENT_LIB} INTERFACE "ld/esp32s2.peripherals.ld")
    target_link_libraries(${COMPONENT_LIB} PUBLIC gcc)
    target_link_libraries(${COMPONENT_LIB} INTERFACE "-u call_user_start_cpu0")

    idf_build_get_property(config_dir CONFIG_DIR)
    # Preprocess esp32s2.ld linker script to include configuration, becomes esp32s2_out.ld
    set(LD_DIR ${CMAKE_CURRENT_SOURCE_DIR}/ld)
    add_custom_command(
        OUTPUT esp32s2_out.ld
        COMMAND "${CMAKE_C_COMPILER}" -C -P -x c -E -o esp32s2_out.ld -I ${config_dir} ${LD_DIR}/esp32s2.ld
        MAIN_DEPENDENCY ${LD_DIR}/esp32s2.ld ${sdkconfig_header}
        COMMENT "Generating linker script..."
        VERBATIM)

    add_custom_target(esp32s2_linker_script DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/esp32s2_out.ld)
    add_dependencies(${COMPONENT_LIB} esp32s2_linker_script)
endif()
