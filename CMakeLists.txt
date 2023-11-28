cmake_minimum_required(VERSION 3.0.0...3.24 FATAL_ERROR)
project(webgpu-backend-wgpu VERSION 1.0.0)

set(WEBGPU_BACKEND "WGPU" CACHE STRING "Backend implementation of WebGPU. Possible values are WGPU and DAWN (it does not matter when using emcmake)")

if (EMSCRIPTEN)

	add_library(webgpu INTERFACE)

	target_include_directories(webgpu INTERFACE
		"${CMAKE_CURRENT_SOURCE_DIR}/include-emscripten"
	)

	# This is used to advertise the flavor of WebGPU that this zip provides
	target_compile_definitions(webgpu INTERFACE WEBGPU_BACKEND_EMSCRIPTEN)

else (EMSCRIPTEN)
    string(TOUPPER ${WEBGPU_BACKEND} WEBGPU_BACKEND_U)
    if (WEBGPU_BACKEND_U STREQUAL "WGPU")

        message(STATUS "Using wgpu-native backend for WebGPU")

        set(WGPU ${CMAKE_CURRENT_SOURCE_DIR})
        if (NOT ARCH)
            set(ARCH ${CMAKE_SYSTEM_PROCESSOR})
            if (ARCH STREQUAL "AMD64")
                set(ARCH "x86_64")
            endif()
        endif()

        # A pre-compiled target (IMPORTED) that is a statically
        # linked library (STATIC, meaning .lib or .a).
        add_library(webgpu STATIC IMPORTED GLOBAL)

        # This is used to advertise the flavor of WebGPU that this zip provides
        target_compile_definitions(webgpu INTERFACE WEBGPU_BACKEND_WGPU)

        if(CMAKE_SYSTEM_NAME STREQUAL "Windows")

            set(WGPU_RUNTIME_LIB ${WGPU}/lib/windows-${ARCH}/wgpu_native.lib)
            target_link_libraries(webgpu INTERFACE "ws2_32" "D3DCompiler" "userenv" "ntdll" "Opengl32" "Bcrypt")
            set_target_properties(
                webgpu
                PROPERTIES
                    IMPORTED_LOCATION "${WGPU_RUNTIME_LIB}"
                    IMPORTED_IMPLIB "${WGPU_RUNTIME_LIB}"
                    INTERFACE_INCLUDE_DIRECTORIES "${WGPU}/include"
            )

        elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")

            set(WGPU_RUNTIME_LIB ${WGPU}/lib/linux-${ARCH}/libwgpu_native.a)
            set_target_properties(
                webgpu
                PROPERTIES
                    IMPORTED_LOCATION "${WGPU_RUNTIME_LIB}"
                    INTERFACE_INCLUDE_DIRECTORIES "${WGPU}/include"
            )

        elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")

            set(WGPU_RUNTIME_LIB ${WGPU}/lib/macos-${ARCH}/libwgpu_native.a)
            target_link_libraries(webgpu INTERFACE "-framework Metal" "-framework QuartzCore" "-framework MetalKit")
            set_target_properties(
                webgpu
                PROPERTIES
                    IMPORTED_LOCATION "${WGPU_RUNTIME_LIB}"
                    INTERFACE_INCLUDE_DIRECTORIES "${WGPU}/include"
            )

        else()

            message(FATAL_ERROR "Plateform not supported by this release of WebGPU. You may consider building it yourself from its source (see https://github.com/gfx-rs/wgpu-native)")

        endif()

        message(STATUS "Using WebGPU backend from '${WGPU_RUNTIME_LIB}'")
        set(WGPU_RUNTIME_LIB ${WGPU_RUNTIME_LIB} PARENT_SCOPE)
        set(WGPU_RUNTIME_LIB ${WGPU_RUNTIME_LIB} CACHE INTERNAL "Path to the WebGPU library binary")

    elseif (WEBGPU_BACKEND_U STREQUAL "DAWN")
        message(STATUS "Using dawn backend for WebGPU")

        add_library(webgpu INTERFACE)
        include(cmake/FetchDawn.cmake)

        target_link_libraries(webgpu INTERFACE webgpu_dawn)
        target_include_directories(webgpu INTERFACE
            "${CMAKE_CURRENT_SOURCE_DIR}/include-dawn"
            "${CMAKE_BINARY_DIR}/_deps/dawn-src/include"
        )

        # This is used to advertise the flavor of WebGPU that this zip provides
        target_compile_definitions(webgpu INTERFACE WEBGPU_BACKEND_DAWN)

	else()
        message(FATAL_ERROR "Invalid value for WEBGPU_BACKEND: possible values are WGPU and DAWN, but '${WEBGPU_BACKEND_U}' was provided.")
    endif()
endif (EMSCRIPTEN)

# Does nothing when statically linked
function(target_copy_webgpu_binaries Target)
endfunction()
