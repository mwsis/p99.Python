cmake_minimum_required(VERSION 3.16)

project(p99_python
    DESCRIPTION "Python bindings for the p99 performance percentile histogram"
    LANGUAGES C
)

find_package(Python3 REQUIRED COMPONENTS Interpreter)

execute_process(
    COMMAND "${Python3_EXECUTABLE}" -c
        "import os, sys, sysconfig\n\
for candidate in (\n\
    sysconfig.get_path('include'),\n\
    os.path.join(sys.prefix, 'include', 'python%d.%d' % sys.version_info[:2]),\n\
    os.path.join(sys.prefix, 'include'),\n\
):\n\
    if os.path.isfile(os.path.join(candidate, 'Python.h')):\n\
        print(candidate)\n\
        break\n"
    OUTPUT_VARIABLE PYTHON_INCLUDE_DIR
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
if(NOT PYTHON_INCLUDE_DIR OR NOT EXISTS "${PYTHON_INCLUDE_DIR}/Python.h")
    message(FATAL_ERROR "Could not locate Python.h for ${Python3_EXECUTABLE}")
endif()
execute_process(
    COMMAND "${Python3_EXECUTABLE}" -c "import sysconfig; print(sysconfig.get_config_var('EXT_SUFFIX'))"
    OUTPUT_VARIABLE PYTHON_EXT_SUFFIX
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
execute_process(
    COMMAND "${Python3_EXECUTABLE}" -c "import sysconfig; print(sysconfig.get_config_var('LIBDIR'))"
    OUTPUT_VARIABLE PYTHON_LIB_DIR
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

add_subdirectory(vendor/p99)

add_library(_ext MODULE src/p99/_ext.c)
target_include_directories(_ext PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/vendor/p99/include"
    "${PYTHON_INCLUDE_DIR}"
)
target_link_libraries(_ext PRIVATE p99_static)
if(APPLE)
    target_link_options(_ext PRIVATE -undefined dynamic_lookup)
else()
    execute_process(
        COMMAND "${Python3_EXECUTABLE}" -c
            "import sysconfig; print(sysconfig.get_config_var('LIBDIR'))"
        OUTPUT_VARIABLE PYTHON_LIB_DIR
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    execute_process(
        COMMAND "${Python3_EXECUTABLE}" -c
            "import sysconfig; print(sysconfig.get_config_var('LDLIBRARY'))"
        OUTPUT_VARIABLE PYTHON_LIB_NAME
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    target_link_directories(_ext PRIVATE "${PYTHON_LIB_DIR}")
    target_link_libraries(_ext PRIVATE "${PYTHON_LIB_NAME}")
endif()
if(UNIX)
    target_link_libraries(_ext PRIVATE m)
endif()

set_target_properties(_ext PROPERTIES
    PREFIX ""
    OUTPUT_NAME "_ext"
    SUFFIX "${PYTHON_EXT_SUFFIX}"
    C_STANDARD 11
    C_STANDARD_REQUIRED ON
    C_EXTENSIONS OFF
)

install(TARGETS _ext LIBRARY DESTINATION p99)
