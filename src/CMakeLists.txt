# Must be included before CMAKE_INSTALL_INCLUDEDIR is used.
include(GNUInstallDirs)

add_library(secp256k1_precomputed OBJECT EXCLUDE_FROM_ALL
  precomputed_ecmult.c
  precomputed_ecmult_gen.c
)

# Add objects explicitly rather than linking to the object libs to keep them
# from being exported.
add_library(secp256k1 secp256k1.c $<TARGET_OBJECTS:secp256k1_precomputed>)

add_library(secp256k1_asm INTERFACE)
if(SECP256K1_ASM STREQUAL "arm")
  add_library(secp256k1_asm_arm OBJECT EXCLUDE_FROM_ALL)
  target_sources(secp256k1_asm_arm PUBLIC
    asm/field_10x26_arm.s
  )
  target_sources(secp256k1 PRIVATE $<TARGET_OBJECTS:secp256k1_asm_arm>)
  target_link_libraries(secp256k1_asm INTERFACE secp256k1_asm_arm)
endif()

# Define our export symbol only for Win32 and only for shared libs.
# This matches libtool's usage of DLL_EXPORT
if(WIN32)
  set_target_properties(secp256k1 PROPERTIES DEFINE_SYMBOL "DLL_EXPORT")
endif()

# Object libs don't know if they're being built for a shared or static lib.
# Grab the PIC property from secp256k1 which knows.
get_target_property(use_pic secp256k1 POSITION_INDEPENDENT_CODE)
set_target_properties(secp256k1_precomputed PROPERTIES POSITION_INDEPENDENT_CODE ${use_pic})

target_include_directories(secp256k1 PUBLIC
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
set_target_properties(secp256k1 PROPERTIES
  VERSION "${${PROJECT_NAME}_LIB_VERSION_CURRENT}.${${PROJECT_NAME}_LIB_VERSION_AGE}.${${PROJECT_NAME}_LIB_VERSION_REVISION}"
  SOVERSION "${${PROJECT_NAME}_LIB_VERSION_CURRENT}"
)

if(SECP256K1_BUILD_BENCHMARK)
  add_executable(bench bench.c)
  target_link_libraries(bench secp256k1)
  add_executable(bench_internal bench_internal.c)
  target_link_libraries(bench_internal secp256k1_precomputed secp256k1_asm)
  add_executable(bench_ecmult bench_ecmult.c)
  target_link_libraries(bench_ecmult secp256k1_precomputed secp256k1_asm)
endif()

if(SECP256K1_BUILD_TESTS)
  add_executable(noverify_tests tests.c)
  target_link_libraries(noverify_tests secp256k1_precomputed secp256k1_asm)
  add_test(noverify_tests noverify_tests)
  if(NOT CMAKE_BUILD_TYPE STREQUAL "Coverage")
    add_executable(tests tests.c)
    target_compile_definitions(tests PRIVATE VERIFY)
    target_link_libraries(tests secp256k1_precomputed secp256k1_asm)
    add_test(tests tests)
  endif()
endif()

if(SECP256K1_BUILD_EXHAUSTIVE_TESTS)
  # Note: do not include secp256k1_precomputed in exhaustive_tests (it uses runtime-generated tables).
  add_executable(exhaustive_tests tests_exhaustive.c)
  target_link_libraries(exhaustive_tests secp256k1_asm)
  target_compile_definitions(exhaustive_tests PRIVATE $<$<NOT:$<CONFIG:Coverage>>:VERIFY>)
  add_test(exhaustive_tests exhaustive_tests)
endif()

if(SECP256K1_BUILD_CTIME_TESTS)
  add_executable(ctime_tests ctime_tests.c)
  target_link_libraries(ctime_tests secp256k1)
endif()

if(SECP256K1_INSTALL)
  install(TARGETS secp256k1
    EXPORT ${PROJECT_NAME}-targets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  )
  set(${PROJECT_NAME}_headers
    "${PROJECT_SOURCE_DIR}/include/secp256k1.h"
    "${PROJECT_SOURCE_DIR}/include/secp256k1_preallocated.h"
  )
  if(SECP256K1_ENABLE_MODULE_ECDH)
    list(APPEND ${PROJECT_NAME}_headers "${PROJECT_SOURCE_DIR}/include/secp256k1_ecdh.h")
  endif()
  if(SECP256K1_ENABLE_MODULE_RECOVERY)
    list(APPEND ${PROJECT_NAME}_headers "${PROJECT_SOURCE_DIR}/include/secp256k1_recovery.h")
  endif()
  if(SECP256K1_ENABLE_MODULE_EXTRAKEYS)
    list(APPEND ${PROJECT_NAME}_headers "${PROJECT_SOURCE_DIR}/include/secp256k1_extrakeys.h")
  endif()
  if(SECP256K1_ENABLE_MODULE_SCHNORRSIG)
    list(APPEND ${PROJECT_NAME}_headers "${PROJECT_SOURCE_DIR}/include/secp256k1_schnorrsig.h")
  endif()
  install(FILES ${${PROJECT_NAME}_headers}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  )

  install(EXPORT ${PROJECT_NAME}-targets
    FILE ${PROJECT_NAME}-targets.cmake
    NAMESPACE ${PROJECT_NAME}::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
  )

  include(CMakePackageConfigHelpers)
  configure_package_config_file(
    ${PROJECT_SOURCE_DIR}/cmake/config.cmake.in
    ${PROJECT_NAME}-config.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
    NO_SET_AND_CHECK_MACRO
  )
  write_basic_package_version_file(${PROJECT_NAME}-config-version.cmake
    COMPATIBILITY SameMajorVersion
  )

  install(
    FILES
      ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake
      ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)
endif()
