#
# Copyright (c) 2016 Alibek Omarov
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
#
cmake_minimum_required(VERSION 3.0.0)
project(XASH_ENGINE C)

set(XASH_ENGINE xash) # Lib name

fwgs_fix_default_msvc_settings()
file(GLOB_RECURSE XASH_ENGINE_SOURCES         common/*.c server/*.c ) # Minimal sources
file(GLOB_RECURSE XASH_ENGINE_WIN32_SOURCES   ../game_launch/game.rc platform/win32/*.c) # Win32 only sources
file(GLOB_RECURSE XASH_ENGINE_SDL_SOURCES     platform/sdl/*.c) # SDL version sources
file(GLOB_RECURSE XASH_ENGINE_POSIX_SOURCES    platform/posix/con_posix.c) # Posix sources
file(GLOB_RECURSE XASH_ENGINE_STUB_SOURCES     platform/stub/*.c) # Stub sources
file(GLOB_RECURSE XASH_ENGINE_ANDROID_SOURCES platform/android/*.c) # Android sources
file(GLOB_RECURSE XASH_ENGINE_CLIENT_SOURCES  client/*.c) # Client sources

if(NOT WIN32 AND NOT APPLE)
	find_package (Libunwind QUIET)
endif()

if(LIBUNWIND_FOUND AND UNIX AND NOT APPLE)
	message(STATUS "libunwind found")
	include_directories(${LIBUNWIND_INCLUDE_DIRS})
	list(APPEND XASH_ENGINE_POSIX_SOURCES platform/posix/crashhandler_libunwind.c)
elseif(UNIX)
	message(STATUS "libunwind not found")
	list(APPEND XASH_ENGINE_POSIX_SOURCES platform/posix/crashhandler.c)
	set(LIBUNWIND_INCLUDE_DIR "")
	set(LIBUNWIND_LIBRARIES "")
elseif(NOT WIN32)
#	file(GLOB XASH_ENGINE_POSIX_SOURCES platform/stub/crashhandler.c)
endif()

cond_list_append(WIN32        XASH_ENGINE_SOURCES XASH_ENGINE_WIN32_SOURCES)
cond_list_append(UNIX        XASH_ENGINE_SOURCES XASH_ENGINE_POSIX_SOURCES)
cond_list_append("NOT UNIX AND NOT WIN32"        XASH_ENGINE_SOURCES XASH_ENGINE_STUB_SOURCES)
cond_list_append(XASH_SDL     XASH_ENGINE_SOURCES XASH_ENGINE_SDL_SOURCES)
cond_list_append(ANDROID      XASH_ENGINE_SOURCES XASH_ENGINE_ANDROID_SOURCES)
cond_list_append("NOT XASH_DEDICATED" XASH_ENGINE_SOURCES XASH_ENGINE_CLIENT_SOURCES)

include_directories(
        ${CMAKE_CURRENT_BINARY_DIR}
	.
	common/
	common/imagelib/
	common/soundlib/
	client/
	client/vgui/
	server/
	../common
	../pm_shared
)

if(XASH_NANOGL)
	file(GLOB_RECURSE XASH_ENGINE_GLESWRAP_SOURCES ${XASH_NANOGL_SRC}/*.cpp) # NanoGL version
	if(NOT XASH_ENGINE_GLESWRAP_SOURCES)
		message(FATAL_ERROR "Clone both nanogl sources to engine folder!")
	endif()
	list(APPEND XASH_ENGINE_SOURCES ${XASH_ENGINE_GLESWRAP_SOURCES})

	include_directories(${XASH_NANOGL_SRC}/ ${XASH_NANOGL_SRC}/GL/)
	add_definitions(-DXASH_NANOGL -D__MULTITEXTURE_SUPPORT__)
elseif(XASH_WES)
	file(GLOB XASH_ENGINE_GLESWRAP_SOURCES ${XASH_WES_SRC}/src/*.c) # wesgl version
	if(NOT XASH_ENGINE_GLESWRAP_SOURCES)
		message(FATAL_ERROR "Clone both gl-wes-v2 and nanogl sources to engine folder!")
	endif()
	list(APPEND XASH_ENGINE_SOURCES ${XASH_ENGINE_GLESWRAP_SOURCES})

	include_directories(${XASH_NANOGL_SRC}/ ${XASH_NANOGL_SRC}/GL/ ${XASH_WES_SRC}/src/)
	add_definitions(-DXASH_WES)
endif()

if(UNIX)
    add_definitions(-DXASH_DYNAMIC_DLADDR)
endif()

add_definitions(-DXASH_FORCEINLINE)

# Important, as HLSDK and engine shares some symbol names!
if(NOT WIN32) # Windows by default have hidden visibility and better linker
	fwgs_add_compile_options(C -fvisibility=hidden)
endif()

# ----- Conditions -----

if(XASH_VECTORIZE_SINCOS) # I know what I am doing and I want to build version that requires SSE
	add_definitions(-DVECTORIZE_SINCOS)
endif()

if(XASH_SINGLE_BINARY) # Set executable or library
	add_definitions(-DSINGLE_BINARY)
	add_executable(${XASH_ENGINE} ${XASH_ENGINE_SOURCES})
	if(MSVC)
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SUBSYSTEM:WINDOWS")
	endif()
else()
	add_library(${XASH_ENGINE} SHARED ${XASH_ENGINE_SOURCES})
endif()

if(LIBUNWIND_FOUND)
	target_link_libraries(${XASH_ENGINE} ${LIBUNWIND_LIBRARIES})
endif()

if(MSVC)
	add_definitions(-D_CRT_SECURE_NO_WARNINGS -D_CRT_NONSTDC_NO_DEPRECATE)
endif()

if(WIN32 AND NOT XASH_64BIT)
	add_definitions(-DDBGHELP) # dbghelp crashhandler
	if(MINGW)
		target_link_libraries(${XASH_ENGINE} -luser32 -lkernel32 -lgdi32 -lws2_32 -ldbghelp -lpsapi)
	elseif(MSVC)
		target_link_libraries(${XASH_ENGINE} user32.lib kernel32.lib gdi32.lib Ws2_32.lib dbghelp.lib psapi.lib)
	endif()
else()
	if(XASH_USE_SELECT)
		add_definitions(-DUSE_SELECT)
	endif()
        if(MINGW)
                target_link_libraries(${XASH_ENGINE} -luser32 -lkernel32 -lgdi32 -lws2_32)
        elseif(MSVC)
                target_link_libraries(${XASH_ENGINE} user32.lib kernel32.lib gdi32.lib Ws2_32.lib)
        endif()
endif()

if(LINUX)
	add_definitions(-DGDB_BREAK)
endif()

if(NOT XASH_64BIT AND NOT MSVC)
	add_definitions(-DXASH_FASTSTR)
endif()

if(XASH_NONSTANDARD_LOAD)
	add_definitions(-DXASH_NONSTANDARD_LOAD)
endif()

if(XASH_DEDICATED)
	add_definitions(-DXASH_DEDICATED)
	if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux" AND NOT XASH_SDL)
		target_link_libraries(${XASH_ENGINE} -lrt)
	endif()
endif()

if(XASH_SDL)
	set(SDL2_BUILDING_EXECUTABLE ${XASH_SINGLE_BINARY}) # link sdl2main, if needed
	xash_link_sdl2(${XASH_ENGINE})
	add_definitions(-DXASH_SDL)
	if(MSVC)
		string(REGEX REPLACE "lib$" "dll" SDL2_DLL "${SDL2_LIBRARY}")
		foreach(path IN LISTS SDL2_DLL)
			# Keep the path that ends with .dll
			if(path MATCHES ".*\\.dll$")
				set(SDL2_DLL "${path}")
			endif()
		endforeach()
		install(FILES ${SDL2_DLL}
			CONFIGURATIONS Debug
			DESTINATION ${LIB_INSTALL_DIR}/Debug/)
		install(FILES ${SDL2_DLL}
			CONFIGURATIONS Release
			DESTINATION ${LIB_INSTALL_DIR}/Release/)
	endif()
endif()

if(XASH_GLES)
	add_definitions(-DXASH_GLES -DXASH_GL_STATIC)
endif()

if(XASH_SAILFISH)
	add_definitions(-D__SAILFISH__)
endif()

if(XASH_DLL_LOADER)
	add_definitions(-DDLL_LOADER)

	include_directories(../loader)

	# See target_link_vgui_hack

	# Not needed anymore, loader is static always
	# add_custom_command(TARGET ${XASH_ENGINE} PRE_LINK COMMAND
	#    ${CMAKE_COMMAND} -E copy $<TARGET_FILE:loader> $<TARGET_FILE_DIR:${XASH_ENGINE}>)

	target_link_libraries(${XASH_ENGINE} loader)
endif()

if(XASH_LIBDL)
	target_link_libraries(${XASH_ENGINE} ${CMAKE_DL_LIBS})
else()
	add_definitions(-DNO_LIBDL)
endif()

if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
	add_definitions(-D_DEBUG)
endif()

if(XASH_RELEASE)
	add_definitions(-DXASH_RELEASE)
else()
	execute_process(COMMAND "git" "rev-parse" "--short" "HEAD"
		WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
		OUTPUT_VARIABLE SHORT_HASH
		OUTPUT_STRIP_TRAILING_WHITESPACE)

	if(SHORT_HASH)
		message(STATUS "git hash: ${SHORT_HASH}")
		add_definitions(-DXASH_BUILD_COMMIT="${SHORT_HASH}")
	else()
		message(AUTHOR_WARNING "git hash: not set")
	endif()
endif()

if(XASH_NO_ASYNC_NS_RESOLVE)
	add_definitions(-DXASH_NO_ASYNC_NS_RESOLVE)
endif()

if(XASH_USE_STB_SPRINTF)
	add_definitions(-DXASH_USE_STB_SPRINTF)
endif()

fwgs_set_default_properties(${XASH_ENGINE})

if(NOT WIN32)
	if(NOT HAIKU)
		target_link_libraries(${XASH_ENGINE} -lm -lpthread)
	else()
		target_link_libraries(${XASH_ENGINE} -lm -lpthread -lnetwork)
	endif()
	# For single binary builds we change name to game_launch's executable name
	if(XASH_SINGLE_BINARY)
		set_target_properties(${XASH_ENGINE} PROPERTIES
			    OUTPUT_NAME "xash3d")
	endif()
else()
	# For Win32 we have split DLL names
	if(XASH_DEDICATED)
		set_target_properties(${XASH_ENGINE} PROPERTIES
			OUTPUT_NAME "xash_dedicated")
	else()
		set_target_properties(${XASH_ENGINE} PROPERTIES
			OUTPUT_NAME "xash_sdl")
	endif()
endif()

fwgs_install(${XASH_ENGINE})
