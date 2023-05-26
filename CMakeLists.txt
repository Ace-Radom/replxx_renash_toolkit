cmake_minimum_required(VERSION 3.5)
project(
	replxx
	# HOMEPAGE_URL "https://github.com/AmokHuginnsson/replxx"
	# DESCRIPTION "replxx - Read Evaluate Print Loop library"
	VERSION 0.0.4
	LANGUAGES CXX C
)

if (NOT DEFINED CMAKE_CXX_STANDARD)
	set(CMAKE_CXX_STANDARD 11)
endif()

if (NOT DEFINED CMAKE_C_STANDARD)
	set(CMAKE_C_STANDARD 99)
endif()

set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)

include(CMakeDependentOption)

find_package(Threads)

cmake_dependent_option(
	REPLXX_BUILD_EXAMPLES
	"Build the examples" ON
	"CMAKE_SOURCE_DIR STREQUAL PROJECT_SOURCE_DIR" OFF
)

set(is-clang $<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:AppleClang>>)
set(is-msvc $<CXX_COMPILER_ID:MSVC>)
set(is-gnu $<CXX_COMPILER_ID:GNU>)
set(compiler-id-clang-or-gnu $<OR:${is-clang},${is-gnu}>)

set(coverage-config $<AND:$<CONFIG:Coverage>,$<OR:${is-gnu},${is-clang}>>)

set(replxx-source-patterns "src/*.cpp" "src/*.cxx")

if (CMAKE_VERSION VERSION_GREATER 3.11)
	list(INSERT replxx-source-patterns 0 CONFIGURE_DEPENDS)
endif()

file(GLOB replxx-sources ${replxx-source-patterns})
set(REPLXX_SRC replxx-sources PARENT_SCOPE)

if(REPLXX_BUILD_ALONE)

	add_library(replxx ${replxx-sources})
	add_library(rena::replxx ALIAS replxx)

	target_include_directories(
		replxx
		PUBLIC
			$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
			$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
		PRIVATE
			$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>
	)
	target_compile_definitions(
		replxx
		PUBLIC
			$<$<NOT:$<BOOL:${BUILD_SHARED_LIBS}>>:REPLXX_STATIC>
			$<$<BOOL:${BUILD_SHARED_LIBS}>:REPLXX_BUILDING_DLL>
		PRIVATE
			$<$<CXX_COMPILER_ID:MSVC>:_CRT_SECURE_NO_WARNINGS=1 /ignore:4503>
	)
	target_compile_options(
		replxx
		PRIVATE
			$<$<AND:$<CONFIG:RelWithDebInfo>,${compiler-id-clang-or-gnu}>:-fomit-frame-pointer>
			$<$<AND:$<CONFIG:MinSizeRel>,${compiler-id-clang-or-gnu}>:-Os>
			$<$<AND:$<CONFIG:Debug>,$<CXX_COMPILER_ID:GNU>>:-g -ggdb -g3 -ggdb3>
			$<${coverage-config}:-O0 --coverage>
			$<${coverage-config}:-fno-inline -fno-default-inline>
			$<${coverage-config}:-fno-inline-small-functions>
			$<${compiler-id-clang-or-gnu}:-Wall -Wextra>
			$<$<CXX_COMPILER_ID:GNU>:-Wno-unknown-pragmas>
	)
	if (NOT CMAKE_VERSION VERSION_LESS 3.13)
		target_link_options(
			replxx
			PRIVATE
				$<$<AND:$<CONFIG:Debug>,$<CXX_COMPILER_ID:GNU>>:-g -ggdb -g3 -ggdb3>
				$<${coverage-config}:--coverage>
				$<${is-msvc}:/ignore:4099>
		)
	else()
		# "safest" way prior to 3.13
		target_link_libraries(
			replxx
			PRIVATE
				$<${coverage-config}:--coverage>
				$<${is-msvc}:/ignore:4099>
		)
	endif()
	target_link_libraries(replxx PUBLIC Threads::Threads)
	set_target_properties(replxx PROPERTIES VERSION ${PROJECT_VERSION})

	set_property(TARGET replxx PROPERTY DEBUG_POSTFIX -d)
	set_property(TARGET replxx PROPERTY RELWITHDEBINFO_POSTFIX -rd)
	set_property(TARGET replxx PROPERTY MINSIZEREL_POSTFIX)
	if (NOT BUILD_SHARED_LIBS AND MSVC)
		set_property(TARGET replxx PROPERTY OUTPUT_NAME replxx-static)
	endif()

	if (REPLXX_BUILD_EXAMPLES)
		add_executable(replxx-example-cxx-api "")

		target_sources(
			replxx-example-cxx-api
			PRIVATE
				examples/cxx-api.cxx
				examples/util.c
		)
		target_compile_definitions(replxx-example-cxx-api 
			PRIVATE
				REPLXX_STATIC
				$<$<CXX_COMPILER_ID:MSVC>:_CRT_SECURE_NO_WARNINGS=1>
		)
		target_link_options(replxx-example-cxx-api
			PRIVATE
				$<$<AND:$<CONFIG:Debug>,$<CXX_COMPILER_ID:GNU>>:-g -ggdb -g3 -ggdb3>
		)
		target_link_libraries(replxx-example-cxx-api 
			PRIVATE 
				rena::replxx
		)
		target_link_libraries(replxx-example-cxx-api
			PRIVATE
				$<${coverage-config}:--coverage>
		)
	endif()

endif()