cmake_minimum_required(VERSION 3.13) # 2.2 - case insensitive syntax
                                     # 3.13 included policy CMP0077

project(ModbusTools VERSION 0.5.0 LANGUAGES CXX)

set(MBTOOLS_VERSION ${PROJECT_VERSION})
set(MBTOOLS_ROOT_DIR ${CMAKE_SOURCE_DIR})

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(MBTOOLS_CLIENT_ENABLED "Enable client application build" ON)
option(MBTOOLS_SERVER_ENABLED "Enable server application build" ON)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/Debug")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_DEBUG "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/Debug")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG "${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}/Debug")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/Release")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_RELEASE "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/Release")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE "${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}/Release")

if (APPLE)
    #set(CMAKE_SKIP_BUILD_RPATH  FALSE)
    set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
    set(CMAKE_INSTALL_RPATH "@executable_path/../Frameworks;@loader_path")
elseif (NOT WIN32)
    #set(CMAKE_SKIP_BUILD_RPATH  FALSE)
    set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
    set(CMAKE_INSTALL_RPATH $ORIGIN)
    #set(CMAKE_INSTALL_RPATH "\${ORIGIN}")
endif()

set(MB_QT_ENABLED ON)
set(MB_C_SUPPORT_DISABLE ON)
set(MB_NO_INSTALL ON)
# EXCLUDE_FROM_ALL: to not generate install data
add_subdirectory(modbus/src EXCLUDE_FROM_ALL)

add_subdirectory(src/core)

if (MBTOOLS_CLIENT_ENABLED)
    add_subdirectory(src/client)
endif()

if (MBTOOLS_SERVER_ENABLED)
    add_subdirectory(src/server)
endif()

add_subdirectory("doc")

install(TARGETS ${MB_LIBRARY_NAME}         DESTINATION .)
install(TARGETS ${MBTOOLS_CORE_LIB_NAME}   DESTINATION .)

if (MBTOOLS_CLIENT_ENABLED)
    if (APPLE)
        install(TARGETS ${MBTOOLS_CLIENT_APP_NAME} BUNDLE DESTINATION . RUNTIME DESTINATION .)
    else()
        install(TARGETS ${MBTOOLS_CLIENT_APP_NAME} DESTINATION .)
    endif()
endif()

if (MBTOOLS_SERVER_ENABLED)
    if (APPLE)
        install(TARGETS ${MBTOOLS_SERVER_APP_NAME} BUNDLE DESTINATION . RUNTIME DESTINATION .)
    else()
        install(TARGETS ${MBTOOLS_SERVER_APP_NAME} DESTINATION .)
    endif()
    # Install scripts
    # Collect Python files using GLOB
    file(GLOB MBTOOLS_SCRIPT_FILES "${CMAKE_SOURCE_DIR}/src/server/python/*.py")
    # Install Python files
    if (APPLE)
        install(FILES ${MBTOOLS_SCRIPT_FILES} DESTINATION "${MBTOOLS_SERVER_APP_NAME}.app/Contents/Resources/script/server")
    else()
        install(FILES ${MBTOOLS_SCRIPT_FILES} DESTINATION "./script/server")
    endif()
endif()

# Install help files
# Collect help files using GLOB
file(GLOB MBTOOLS_HELP_FILES "${CMAKE_BINARY_DIR}/doc/output/*.qch" "${CMAKE_BINARY_DIR}/doc/output/*.qhc")
if (APPLE)
    if (MBTOOLS_CLIENT_ENABLED)
        install(FILES ${MBTOOLS_HELP_FILES} DESTINATION "${MBTOOLS_CLIENT_APP_NAME}.app/Contents/Resources/help")
    endif()
    if (MBTOOLS_SERVER_ENABLED)
        install(FILES ${MBTOOLS_HELP_FILES} DESTINATION "${MBTOOLS_SERVER_APP_NAME}.app/Contents/Resources/help")
    endif()
else()
    install(FILES ${MBTOOLS_HELP_FILES} DESTINATION "./help")
endif()

if (APPLE)
    find_program(MBTOOLS_MACDEPLOYQT_EXECUTABLE
                 NAMES macdeployqt
                 HINTS "/usr/local/opt/qt@5/bin" "/opt/homebrew/opt/qt@5/bin")

    if (MBTOOLS_MACDEPLOYQT_EXECUTABLE)
        set(MBTOOLS_MACDEPLOY_TARGETS)
        set(MBTOOLS_MACDEPLOY_DEPENDS)

        if (MBTOOLS_CLIENT_ENABLED)
            list(APPEND MBTOOLS_MACDEPLOY_TARGETS
                COMMAND "${MBTOOLS_MACDEPLOYQT_EXECUTABLE}" "$<TARGET_BUNDLE_DIR:${MBTOOLS_CLIENT_APP_NAME}>"
                        -always-overwrite
                        "-libpath=$<TARGET_FILE_DIR:${MBTOOLS_CORE_LIB_NAME}>")
            list(APPEND MBTOOLS_MACDEPLOY_DEPENDS ${MBTOOLS_CLIENT_APP_NAME})
        endif()

        if (MBTOOLS_SERVER_ENABLED)
            list(APPEND MBTOOLS_MACDEPLOY_TARGETS
                COMMAND "${MBTOOLS_MACDEPLOYQT_EXECUTABLE}" "$<TARGET_BUNDLE_DIR:${MBTOOLS_SERVER_APP_NAME}>"
                        -always-overwrite
                        "-libpath=$<TARGET_FILE_DIR:${MBTOOLS_CORE_LIB_NAME}>")
            list(APPEND MBTOOLS_MACDEPLOY_DEPENDS ${MBTOOLS_SERVER_APP_NAME})
        endif()

        add_custom_target(deploy-macos
            ${MBTOOLS_MACDEPLOY_TARGETS}
            DEPENDS ${MBTOOLS_MACDEPLOY_DEPENDS}
            COMMENT "Deploy Qt frameworks into macOS application bundles"
        )
    else()
        message(STATUS "MBTOOLS: macdeployqt was not found; deploy-macos target will not be generated")
    endif()
endif()
