# Tencent is pleased to support the open source community by making xLua available.
# Copyright (C) 2016 THL A29 Limited, a Tencent company. All rights reserved.
# Licensed under the MIT License (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at
# http://opensource.org/licenses/MIT
# Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

cmake_minimum_required(VERSION 2.8)

if ( WIN32 AND NOT CYGWIN AND NOT ( CMAKE_SYSTEM_NAME STREQUAL "WindowsStore" ) AND NOT ANDROID)
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MT" CACHE STRING "")
	set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /MTd" CACHE STRING "")
	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT" CACHE STRING "")
	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd" CACHE STRING "")
endif ()

project(PuertsPlugin)

set (CMAKE_CXX_STANDARD 14)

if ( IOS )
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fembed-bitcode")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fembed-bitcode")
endif ()


find_path(PUERTS_PROJECT_DIR NAMES SConstruct
    PATHS 
    ${CMAKE_SOURCE_DIR}
    NO_DEFAULT_PATH
    )

MARK_AS_ADVANCED(PUERTS_PROJECT_DIR)

set(V8_ENGINE ${CMAKE_SOURCE_DIR}/v8)

set(V8_INC ${V8_ENGINE}/Inc)

set(ThirdParty ${CMAKE_SOURCE_DIR}/../../unreal/Puerts/ThirdParty)
set(WEBSOCKET_INC ${ThirdParty}/Include/websocketpp)
set(ASIO_INC ${ThirdParty}/Include/asio)

include_directories(
    ${CMAKE_SOURCE_DIR}
    ${V8_INC}
    ${WEBSOCKET_INC}
    ${ASIO_INC}
    Inc
    ${CMAKE_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private
)

set ( PUERTS_INC
    Inc/Log.h
    Inc/JSEngine.h
    Inc/V8Utils.h
    Inc/JSFunction.h
    ${CMAKE_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private/V8InspectorImpl.h
    ${CMAKE_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private/PromiseRejectCallback.hpp
)

set ( PUERTS_SRC
    Src/Puerts.cpp
    Src/Log.cpp
    Src/JSEngine.cpp
    Src/JSFunction.cpp
    ${CMAKE_SOURCE_DIR}/../../unreal/Puerts/Source/JsEnv/Private/V8InspectorImpl.cpp
)

macro(source_group_by_dir proj_dir source_files)
    if(MSVC OR APPLE)
        get_filename_component(sgbd_cur_dir ${proj_dir} ABSOLUTE)
        foreach(sgbd_file ${${source_files}})
			get_filename_component(sgbd_abs_file ${sgbd_file} ABSOLUTE)
            file(RELATIVE_PATH sgbd_fpath ${sgbd_cur_dir} ${sgbd_abs_file})
            string(REGEX REPLACE "\(.*\)/.*" \\1 sgbd_group_name ${sgbd_fpath})
            string(COMPARE EQUAL ${sgbd_fpath} ${sgbd_group_name} sgbd_nogroup)
            string(REPLACE "/" "\\" sgbd_group_name ${sgbd_group_name})
            if(sgbd_nogroup)
                set(sgbd_group_name "\\")
            endif(sgbd_nogroup)
            source_group(${sgbd_group_name} FILES ${sgbd_file})
        endforeach(sgbd_file)
    endif(MSVC OR APPLE)
endmacro(source_group_by_dir)

source_group_by_dir(${CMAKE_CURRENT_SOURCE_DIR} PUERTS_INC)
source_group_by_dir(${CMAKE_CURRENT_SOURCE_DIR} PUERTS_SRC)

if (APPLE)
    if (IOS)
        set(CMAKE_OSX_ARCHITECTURES "$(ARCHS_STANDARD)")
        add_library(puerts STATIC
           ${PUERTS_SRC} ${PUERTS_INC}
        )
		set_xcode_property (puerts IPHONEOS_DEPLOYMENT_TARGET "7.0" "all")
    else ()
        set(CMAKE_OSX_ARCHITECTURES "$(ARCHS_STANDARD_64_BIT)")
        add_library(puerts MODULE
            ${PUERTS_SRC} ${PUERTS_INC}
        )
        set_target_properties ( puerts PROPERTIES BUNDLE TRUE )
    endif ()
else ( )
    add_library(puerts SHARED
        ${PUERTS_SRC} ${PUERTS_INC}
    )
endif ( )

if ( WIN32 AND NOT CYGWIN )
    target_compile_definitions (puerts PRIVATE BUILDING_V8_SHARED)
endif ( )

if ( WIN32 )
    if("${CMAKE_SIZEOF_VOID_P}" STREQUAL "4")
        target_link_libraries(puerts
            ${V8_ENGINE}/Lib/Win32/wee8.lib
            winmm.lib
            dbghelp.lib
            shlwapi.lib
        )
    else ()
        target_link_libraries(puerts
            ${V8_ENGINE}/Lib/Win64/wee8.lib
            winmm.lib
            dbghelp.lib
            shlwapi.lib
        )
    endif ()
    target_compile_definitions (puerts PRIVATE PLATFORM_WINDOWS)
elseif ( ANDROID )
    find_library(log-lib log )

    if( ANDROID_ABI STREQUAL "armeabi-v7a")
        target_link_libraries(puerts
            ${V8_ENGINE}/Lib/Android/armv7/libwee8.a
            ${log-lib}
        )
        target_compile_definitions (puerts PRIVATE PLATFORM_ANDROID_ARM)
    else ( )
        target_link_libraries(puerts
            ${V8_ENGINE}/Lib/Android/arm64/libwee8.a
            ${log-lib}
        )
        target_compile_definitions (puerts PRIVATE PLATFORM_ANDROID_ARM64)
    endif ( )

    #set_target_properties( puerts PROPERTIES LINK_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/android_version.script)
    #set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--gc-sections -Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/android_version.script")
    #set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--gc-sections")
    add_custom_command(TARGET puerts POST_BUILD
            COMMAND "${ANDROID_TOOLCHAIN_PREFIX}strip" -g -S -d --strip-debug --verbose
            "${CMAKE_BINARY_DIR}/libpuerts.so"
            COMMENT "Strip debug symbols done on final binary.")
elseif ( APPLE )
    if (IOS)
        target_compile_definitions (puerts PRIVATE PLATFORM_IOS)
    else ()
        target_link_libraries(puerts
            ${V8_ENGINE}/Lib/macOS/libwee8.a
        )
        target_compile_definitions (puerts PRIVATE PLATFORM_MAC)
    endif ( )
endif ( )
