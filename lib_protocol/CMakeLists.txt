cmake_minimum_required(VERSION 2.8)
#set(CMAKE_BUILD_TYPE Release)
#set(CMAKE_BUILD_TYPE Release CACHE STRING "set build type to release")

if (${CMAKE_BINARY_DIR} STREQUAL ${CMAKE_CURRENT_SOURCE_DIR})
	message(FATAL_ERROR "Please into another dir to build!")
endif()

#string(TOUPPER ${CMAKE_SYSTEM_NAME} CMAKE_SYSTEM_NAME)
if (CMAKE_SYSTEM_NAME MATCHES "Android")
	add_definitions("-flto")
	add_definitions("-DANDROID")
	add_definitions("-Wno-invalid-source-encoding")
    add_definitions("-Wno-unused-command-line-argument")
    string(APPEND CMAKE_C_FLAGS " -Qunused-arguments")
	set(UNIX_OS true)
elseif (CMAKE_SYSTEM_NAME MATCHES "Linux")
	add_definitions("-Wno-invalid-source-encoding")
	add_definitions("-O2")
	set(UNIX_OS true)
elseif (CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
	add_definitions("-O2")
	set(UNIX_OS true)
elseif (CMAKE_SYSTEM_NAME MATCHES "Darwin")
	add_definitions("-Wno-invalid-source-encoding")
	add_definitions("-Os")
	set(UNIX_OS true)
elseif (CMAKE_SYSTEM_NAME MATCHES "Windows")
	set(WIN_OS true)
else()
	message(FATAL_ERROR "unknown CMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}")
endif()

##############################################################################

set(acl_path ${CMAKE_CURRENT_SOURCE_DIR}/../lib_acl)
set(acl_include ${acl_path}/include)

include_directories(
	${acl_include}
	${CMAKE_CURRENT_SOURCE_DIR}/src
	${CMAKE_CURRENT_SOURCE_DIR}/include
)

set(src ${CMAKE_CURRENT_SOURCE_DIR}/src)
set(sources
	${src}
	${src}/http
	${src}/smtp
	${src}/icmp
)

foreach (iter ${sources})
	aux_source_directory(${iter} lib_src)
endforeach()

##############################################################################

if (${UNIX_OS})
	add_definitions(
		"-W"
		"-fPIC"
		"-Wall"
		"-Werror"
		"-Wshadow"
		"-Wpointer-arith"
		"-Waggregate-return"
		"-Wmissing-prototypes"
		"-D_REENTRANT"
		"-D_USE_FAST_MACRO"
		"-DACL_WRITEABLE_CHECK"
		"-Wno-long-long"
		"-Wuninitialized"
		"-D_POSIX_PTHREAD_SEMANTICS"
		"-DACL_PREPARE_COMPILE"
		"-Wstrict-prototypes"
		"-fvisibility=hidden"
		)

#	if (CMAKE_BUILD_TYPE STREQUAL "")
#		set(CMAKE_BUILD_TYPE "DEBUG")
#	else()
#		string(TOUPPER CMAKE_BUILD_TYPE CMAKE_BUILD_TYPE)
#	endif()

	if (!CMAKE_BUILD_TYPE STREQUAL "RELEASE")
		add_definitions("-g")
	endif()

elseif (${WIN_OS})
	add_definitions(
		"-Yc"
		"-D_WINSOCK_DEPRECATED_NO_WARNINGS"
		"-DACL_WRITEABLE_CHECK"
		"-DACL_PREPARE_COMPILE")
endif()

if (CMAKE_SYSTEM_NAME MATCHES "Android")
	set(CMAKE_SHARED_LINKER_FLAGS "-shared -flto")

	set(lib_output_path ${CMAKE_CURRENT_SOURCE_DIR}/../android/lib/${ANDROID_ABI})
	set(LIBRARY_OUTPUT_PATH ${lib_output_path})
	set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${lib_output_path})

	add_library(protocol_static STATIC ${lib_src})
	SET_TARGET_PROPERTIES(protocol_static PROPERTIES OUTPUT_NAME "protocol")
	link_directories(${lib_output_path})
else()
	set(lib_output_path ${PROJECT_BINARY_DIR}/lib)
	set(LIBRARY_OUTPUT_PATH ${lib_output_path})
	set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${lib_output_path})

	if (${UNIX_OS})
		set(CMAKE_SHARED_LINKER_FLAGS "-shared -Wl,-rpath,. -L${lib_output_path} -lacl")
	elseif (${WIN_OS})
		set(CMAKE_SHARED_LINKER_FLAGS "/DYNAMICBASE ws2_32.lib")
	endif()

	add_library(protocol_static STATIC ${lib_src})
	SET_TARGET_PROPERTIES(protocol_static PROPERTIES OUTPUT_NAME "protocol")
	link_directories(${lib_output_path})
endif()

if (NOT CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
	add_library(protocol SHARED ${lib_src})
	target_link_libraries(protocol acl)
endif()

##############################################################################
