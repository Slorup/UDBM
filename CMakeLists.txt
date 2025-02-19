cmake_minimum_required(VERSION 3.16)
project(UDBM VERSION 2.0.11 LANGUAGES CXX C)
include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

option(TESTING OFF)
option(STATIC OFF)
option(UBSAN OFF)
option(ASAN OFF)

cmake_policy(SET CMP0048 NEW) # project() command manages VERSION variables
set(CMAKE_CXX_STANDARD 17)
set(UDBM_PACKAGE_STRING "${PACKAGE_NAME} ${PACKAGE_VERSION}")
set(UDBM_VERSION "${PACKAGE_VERSION}")
set(ENABLE_STORE_MINGRAPH 1)
CONFIGURE_FILE("src/config.h.cmake" "include/dbm/config.h")

if (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(-Wpedantic -Wall -Wextra)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    add_compile_options(/W4)
endif()

set(CMAKE_PREFIX_PATH "${CMAKE_PREFIX_PATH};${CMAKE_CURRENT_SOURCE_DIR}/libs")
find_package(xxHash 0.8.0 CONFIG REQUIRED)
find_package(UUtils 1.1.1 REQUIRED COMPONENTS base hash debug)

add_library(UDBM src/DBMAllocator.cpp src/dbm.c src/fed_dbm.cpp src/mingraph.c src/mingraph_read.c src/partition.cpp src/print.cpp src/gen.c src/mingraph_cache.cpp src/mingraph_relation.c src/pfed.cpp src/fed.cpp src/infimum.cpp src/mingraph_equal.c src/mingraph_write.c src/priced.cpp)
target_link_libraries(UDBM UUtils::base UUtils::udebug UUtils::hash)

if (ASAN)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address -fno-omit-frame-pointer")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address -fno-omit-frame-pointer")
    set(CMAKE_LINKER_FLAGS "${CMAKE_LINKER_FLAGS} -fsanitize=address -fno-omit-frame-pointer")
endif()
if (UBSAN)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-omit-frame-pointer -fsanitize=undefined")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-omit-frame-pointer -fsanitize=undefined")
    set(CMAKE_LINKER_FLAGS "${CMAKE_LINKER_FLAGS} -fno-omit-frame-pointer -fsanitize=undefined")
endif()


if(STATIC)
	set(CMAKE_CXX_STANDARD_LIBRARIES "-static-libgcc -static-libstdc++ -lwsock32 -lws2_32 ${CMAKE_CXX_STANDARD_LIBRARIES}")
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-Bstatic,--whole-archive -lwinpthread -Wl,--no-whole-archive")
endif(STATIC)

if (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(-Werror=vla)
endif()

target_include_directories(UDBM
    PRIVATE
        # where the library itself will look for its internal headers
        ${CMAKE_CURRENT_SOURCE_DIR}/src
        ${CMAKE_CURRENT_BINARY_DIR}/include
    PUBLIC
        # where top-level project will look for the library's public headers
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        # where external projects will look for the library's public headers
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

include_directories(${CMAKE_CURRENT_BINARY_DIR}/include)

if(TESTING)
    enable_testing()
    add_subdirectory("test")
endif(TESTING)

write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/UDBMConfigVersion.cmake VERSION ${PACKAGE_VERSION} COMPATIBILITY SameMajorVersion)


install(DIRECTORY include DESTINATION .)
install(TARGETS UDBM EXPORT UDBMConfig LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(EXPORT UDBMConfig DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/UDBM/)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/UDBMConfigVersion.cmake DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/UDBM )
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/include/dbm/config.h DESTINATION include/dbm/)
