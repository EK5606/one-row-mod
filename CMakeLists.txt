if(CONFIG_ZMK_BEHAVIOR_HRM_HOLD_TAP)
  zephyr_library()
  zephyr_library_sources(src/behavior_hrm_hold_tap.c)
endif()
