if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
	# Protop: run cmake like this: cmake -G "<generator>" -H. -Bbuild
	message(FATAL_ERROR "Prevented in-tree build.")
endif()

cmake_minimum_required(VERSION 2.8.12)

include(ExternalProject)

project(megasource)

set(MEGA TRUE)

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
	set(MEGA_X64 TRUE)
	set(MEGA_ARCH x64)
else()
	set(MEGA_X86 TRUE)
	set(MEGA_ARCH x86)
endif()

if(MSVC)
	set(CMAKE_SUPPRESS_REGENERATION TRUE)
endif()

if(MSVC)
	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MP")
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MP")
	if(MEGA_X86)
		# This prevents usage of SSE2.
		set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /arch:IA32")
		set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /arch:IA32")
	endif()
	message(STATUS "CMAKE_CXX_FLAGS_RELEASE: ${CMAKE_CXX_FLAGS_RELEASE}")
	message(STATUS "CMAKE_C_FLAGS_RELEASE: ${CMAKE_C_FLAGS_RELEASE}")
endif()

function(add_move_dll ARG_POST_TARGET ARG_TARGET)
	add_custom_command(TARGET ${ARG_POST_TARGET} POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy
		$<TARGET_FILE:${ARG_TARGET}>
		${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>/$<TARGET_FILE_NAME:${ARG_TARGET}>)
endfunction()

function(add_move_file ARG_POST_TARGET ARG_FILE)
	get_filename_component(TEMP_FILENAME ${ARG_FILE} NAME)
	add_custom_command(TARGET ${ARG_POST_TARGET} POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy
		${ARG_FILE}
		${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>/${TEMP_FILENAME})
endfunction()

function(disable_warnings ARG_TARGET)
	get_target_property(OLD_FLAGS ${ARG_TARGET} COMPILE_FLAGS)
	set(NEW_FLAGS -W0)
	if(OLD_FLAGS)
		set(NEW_FLAGS "${OLD_FLAGS} ${NEW_FLAGS}")
	endif()
	set_target_properties(${ARG_TARGET} PROPERTIES COMPILE_FLAGS ${NEW_FLAGS})
endfunction()

# Look for dynamic runtime DLLs.
if(MSVC)
	if(MSVC14)
		set(VSVERSION "140")
	elseif(MSVC12)
		set(VSVERSION "120")
	elseif(MSVC11 OR MSVC10)
		message(WARNING "Visual Studio 2013 (VS12) or newer is required!")
	else()
		message(WARNING "Cannot determine Visual Studio version!")
	endif()

	set(VSCOMNTOOLSSTRING "VS${VSVERSION}COMNTOOLS")
	set(VSCOMNTOOLS "$ENV{${VSCOMNTOOLSSTRING}}")

	if(VSCOMNTOOLS STREQUAL "")
		message(WARNING "Environment variable VSCOMNTOOLS not defined. Is Visual Studio properly installed?")
	else()
		get_filename_component(VS_REDIST_DIR ${VSCOMNTOOLS}/../../VC/redist/${MEGA_ARCH}/Microsoft.VC${VSVERSION}.CRT ABSOLUTE)
		set(MSVCP_DLL ${VS_REDIST_DIR}/msvcp${VSVERSION}.dll)
		if(MSVC12)
			set(MSVCR_DLL ${VS_REDIST_DIR}/msvcr${VSVERSION}.dll)
		else()
			set(MSVCR_DLL ${VS_REDIST_DIR}/vcruntime${VSVERSION}.dll)
			# TODO: add ucrtbase.dll from Program Files (x86)\Windows Kits\10\Redist\ucrt\ ($(UniversalCRT_LibraryPath_${MEGA_ARCH}))
		endif()

		if (EXISTS ${MSVCP_DLL} AND EXISTS ${MSVCR_DLL})
			message(STATUS "Found Visual C and C++ runtime .dll files")
			set(MEGA_MSVC_DLLS ${MSVCP_DLL} ${MSVCR_DLL})
		else()
			message(WARNING "Could not find Visual C and C++ runtime .dll files")
		endif()

		# FIXME: Find some way to avoid hard-coding the version and paths here...
		if(MSVC14)
			get_filename_component(UCRT_DIR "${VSCOMNTOOLS}/../../../Windows Kits/10" ABSOLUTE)
			if(EXISTS ${UCRT_DIR})
				set(UCRT_DLL ${UCRT_DIR}/Redist/ucrt/DLLs/${MEGA_ARCH}/ucrtbase.dll)
				if(EXISTS ${UCRT_DLL})
					set(MEGA_MSVC_DLLS ${MEGA_MSVC_DLLS} ${UCRT_DLL})
					message(STATUS "Found universal C runtime .dll")
				else()
					message(STATUS "Could not find universal C runtime .dll")
				endif()
			else()
				message(STATUS "Could not find universal C runtime folder")
			endif()
		endif()

		# Find vcvarsall.bat.
		get_filename_component(VS_VCVARSALL_BAT ${VSCOMNTOOLS}/../../VC/vcvarsall.bat ABSOLUTE)

		if(EXISTS ${VS_VCVARSALL_BAT})
			message(STATUS "Found vcvarsall.bat")
			set(MEGA_MSVC_VCVARSALL_BAT ${VS_VCVARSALL_BAT})

			if(MEGA_X86)
				set(MEGA_MSVC_VCVARSALL_BAT_ARG "x86")
			else()
				set(MEGA_MSVC_VCVARSALL_BAT_ARG "x86_amd64")
			endif()
		else()
			message(WARNING "Could not find vsvarsall.bat")
		endif()
	endif()
endif()


set(MEGA_ZLIB_VER "1.2.8")
set(MEGA_PHYSFS_VER "2.0.3")
set(MEGA_LUA51_VER "5.1.5")
set(MEGA_LUAJIT_VER "2.0.4")
set(MEGA_LIBOGG_VER "1.3.2")
set(MEGA_LIBVORBIS_VER "1.3.5")
set(MEGA_LIBTHEORA_VER "1.1.1")
set(MEGA_MPG123_VER "1.15.3")
set(MEGA_FREETYPE_VER "2.6.5")
set(MEGA_SDL2_VER "2.0.5-b48d8a98e261")
set(MEGA_OPENAL_VER "1.17.0")
set(MEGA_MODPLUG_VER "0.8.8.4")

set(SKIP_INSTALL_ALL TRUE)

set(ZLIB_FOUND TRUE)
set(ZLIB_LIBRARY zlibstatic)
set(ZLIB_INCLUDE_DIR .)

set(OGG_LIBRARY ogg-static)

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: zlib ${MEGA_ZLIB_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/zlib-${MEGA_ZLIB_VER}" ${CMAKE_BINARY_DIR}/zlib)
set(MEGA_ZLIB zlibstatic)

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: physfs ${MEGA_PHYSFS_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/physfs-${MEGA_PHYSFS_VER}" ${CMAKE_BINARY_DIR}/physfs)
set(MEGA_PHYSFS physfs-static)

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: lua ${MEGA_LUA51_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/lua-${MEGA_LUA51_VER}" ${CMAKE_BINARY_DIR}/lua51)
set(MEGA_LUA51 lua51)

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: libogg ${MEGA_LIBOGG_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/libogg-${MEGA_LIBOGG_VER}" ${CMAKE_BINARY_DIR}/libogg)
set(MEGA_LIBOGG ogg-static)

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: libvorbis ${MEGA_LIBVORBIS_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/libvorbis-${MEGA_LIBVORBIS_VER}" ${CMAKE_BINARY_DIR}/libvorbis)
set(MEGA_LIBVORBIS vorbis-static)
set(MEGA_LIBVORBISFILE vorbisfile-static)

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: libtheora ${MEGA_LIBTHEORA_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/libtheora-${MEGA_LIBTHEORA_VER}" ${CMAKE_BINARY_DIR}/libtheora)
set(MEGA_LIBTHEORA theora-static)

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: mpg123 ${MEGA_MPG123_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/mpg123-${MEGA_MPG123_VER}" ${CMAKE_BINARY_DIR}/mpg123)
set(MEGA_MPEG123 mpg123)

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: freetype ${MEGA_FREETYPE_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/freetype-${MEGA_FREETYPE_VER}" ${CMAKE_BINARY_DIR}/freetype)
set(MEGA_FREETYPE freetype)

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: SDL2 ${MEGA_SDL2_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/SDL2" ${CMAKE_BINARY_DIR}/SDL2)
set(MEGA_SDL2 SDL2)
set(MEGA_SDL2MAIN SDL2main)

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: openal-soft ${MEGA_OPENAL_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/openal-soft" ${CMAKE_BINARY_DIR}/openal-soft)
set(MEGA_OPENAL ${OPENAL_LIB_NAME})

message(STATUS "-----------------------------------------------------")
message(STATUS "Configuring: libmodplug ${MEGA_MODPLUG_VER}")
message(STATUS "-----------------------------------------------------")
add_subdirectory("libs/libmodplug-${MEGA_MODPLUG_VER}" ${CMAKE_BINARY_DIR}/libmodplug)
set(MEGA_MODPLUG modplug-static)

# LuaJIT
if(MSVC)
	message(STATUS "-----------------------------------------------------")
	message(STATUS "Configuring: LuaJIT ${MEGA_LUAJIT_VER}")
	message(STATUS "-----------------------------------------------------")
	set(MEGA_LUAJIT_DIR "LuaJIT")

	# Copy LuaJIT source to binary dir. LuaJIT builds in-source,
	# and we don't want to modify the original source tree, so
	# we copy it.
	file(COPY libs/${MEGA_LUAJIT_DIR} DESTINATION libs)

	set(MEGA_LUAJIT_DIR "LuaJIT")

	set(MEGA_LUAJIT_BUILD_BAT "
		@echo off
		call \"${MEGA_MSVC_VCVARSALL_BAT}\" ${MEGA_MSVC_VCVARSALL_BAT_ARG}
		cd src
		msvcbuild.bat
	")

	set(MEGA_LUAJIT_SOURCE_DIR ${CMAKE_BINARY_DIR}/libs/${MEGA_LUAJIT_DIR})
	file(WRITE ${MEGA_LUAJIT_SOURCE_DIR}/megabuild.bat ${MEGA_LUAJIT_BUILD_BAT})

	ExternalProject_Add(luajit
		SOURCE_DIR ${MEGA_LUAJIT_SOURCE_DIR}
		BUILD_IN_SOURCE 1
		CONFIGURE_COMMAND ""
		BUILD_COMMAND "${MEGA_LUAJIT_SOURCE_DIR}/megabuild.bat"
		INSTALL_COMMAND cmake -E make_directory ${CMAKE_BINARY_DIR}/bin
			COMMAND cmake -E copy ${MEGA_LUAJIT_SOURCE_DIR}/src/lua51.dll ${CMAKE_BINARY_DIR}/bin
			COMMAND cmake -E copy ${MEGA_LUAJIT_SOURCE_DIR}/src/lua51.lib ${CMAKE_BINARY_DIR}/bin)

	set(MEGA_LUAJIT_LIB ${CMAKE_BINARY_DIR}/bin/lua51.lib)
	set(MEGA_LUAJIT_DLL ${CMAKE_BINARY_DIR}/bin/lua51.dll)
	set(MEGA_LUAJIT_INCLUDE ${MEGA_LUAJIT_SOURCE_DIR}/src)

endif()

set(MEGA_3P
	${MEGA_ZLIB}
	${MEGA_PHYSFS}
	${MEGA_LUA51}
	${MEGA_LIBOGG}
	${MEGA_LIBVORBIS}
	${MEGA_LIBVORBISFILE}
	${MEGA_LIBTHEORA}
	${MEGA_MPEG123}
	${MEGA_FREETYPE}
	${MEGA_SDL2}
	${MEGA_SDL2MAIN}
	${MEGA_OPENAL}
	${MEGA_MODPLUG}
)

# Set to directory of love source. Defaults to ./libs/love.
if(NOT MEGA_LOVE)
	set(MEGA_LOVE ${CMAKE_CURRENT_SOURCE_DIR}/libs/love)
endif()

if(EXISTS ${MEGA_LOVE}/CMakeLists.txt)
	message(STATUS "-----------------------------------------------------")
	message(STATUS "Configuring: love")
	message(STATUS "-----------------------------------------------------")
	add_subdirectory(${MEGA_LOVE} ${CMAKE_BINARY_DIR}/love)

	# Turn off third-party warnings when building with LOVE.
	foreach(MEGA_LIB ${MEGA_3P})
		disable_warnings(${MEGA_LIB})
	endforeach()
else()
	message(STATUS "LOVE was not found")
endif()

link_directories(${SDL_LINK_DIR})
add_executable(megatest src/test.cpp)
target_link_libraries(megatest ${MEGA_3P})

add_move_dll(megatest ${MEGA_MPEG123})
add_move_dll(megatest ${MEGA_SDL2})
add_move_dll(megatest ${MEGA_OPENAL})
add_move_dll(megatest ${MEGA_LUA51})
