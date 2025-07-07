if (SKBUILD)
    # invoked as a part of scikit-build, so this is just a proxy for the python
    # extension cmake. this works around the fundamental limitation in cmake
    # that it looks only for directories with a CMakeLists.txt in it, not for a
    # named file
    include(setup-CMakeLists.txt)
    return ()
endif ()

cmake_minimum_required(VERSION 3.15)
project(segyio-python)

if (REQUIRE_PYTHON)
    set(BUILD_PYTHON ON)
endif()

if (NOT BUILD_PYTHON)
    return()
endif()

find_package(Python COMPONENTS Interpreter Development.Module REQUIRED)

python_add_library(_segyio MODULE WITH_SOABI segyio/segyio.cpp)
target_link_libraries(_segyio PRIVATE segyio::segyio)

set(SEGYIO_PYTHON_INSTALL_DIR lib/python${Python_VERSION_MAJOR}.${Python_VERSION_MINOR}/site-packages/segyio)


if (MSVC)
    target_compile_options(_segyio
        BEFORE
        PRIVATE
            /EHsc
    )
endif ()

install(TARGETS _segyio LIBRARY DESTINATION ${SEGYIO_PYTHON_INSTALL_DIR})

# Explicitly install Python files to avoid copying files that are not required,
# e.g., segyio.cpp or folder like __pycache__ that exist if the tests were run
# via ctest.
install(
    FILES
        segyio/__init__.py
        segyio/binfield.py
        segyio/create.py
        segyio/depth.py
        segyio/field.py
        segyio/gather.py
        segyio/line.py
        segyio/open.py
        segyio/segy.py
        segyio/segysampleformat.py
        segyio/tools.py
        segyio/trace.py
        segyio/tracefield.py
        segyio/tracesortingformat.py
        segyio/utils.py
    DESTINATION ${SEGYIO_PYTHON_INSTALL_DIR}
)
install(
    FILES
        segyio/su/__init__.py
        segyio/su/file.py
        segyio/su/words.py
    DESTINATION ${SEGYIO_PYTHON_INSTALL_DIR}/su
)

# Emulate in-place build behavior of invoking `python setup.py build_ext -i`.
# This places the binding library into the `python/segyio` directory. Doing so
# allows the usage of `ctest` to also test the Python package.
if (MSVC)
    # On Windows, setting the target properties does not work.
    add_custom_command(TARGET _segyio POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:_segyio> ${CMAKE_CURRENT_SOURCE_DIR}/segyio
    )
else()
    set_target_properties(_segyio PROPERTIES
        LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/segyio
    )
endif()

option(BUILD_PYDOC "Build python documentation" OFF)

if(BUILD_DOC)
    set(BUILD_PYDOC ON)
endif()

if(BUILD_PYDOC)
    add_custom_target(pydoc
            # run sphinx as the same python version that was just built
            # otherwise, the it will search for the wrong segyio extension and
            # fail
        COMMAND ${Python_EXECUTABLE} -m sphinx
            # use the -d argument to avoid putting cache dir in docs/, because
            # that directory will be install'd
            -d ${CMAKE_CURRENT_BINARY_DIR}/.doctrees
            ${SPHINX_ARGS}
            ${CMAKE_CURRENT_SOURCE_DIR}/docs
            ${CMAKE_CURRENT_BINARY_DIR}/docs
        DEPENDS docs/conf.py
                docs/index.rst
                docs/segyio.rst
        COMMENT "Building python documentation with sphinx"
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    )
    add_dependencies(doc pydoc)
    add_dependencies(pydoc segyio-python)

    install(
        DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/docs/
        DESTINATION ${CMAKE_INSTALL_DATADIR}/doc/segyio
    )
endif()

if(NOT BUILD_TESTING)
    return()
endif()

file(MAKE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/test-artifacts)

add_test(
    NAME python.unit
    COMMAND ${Python_EXECUTABLE} -m pytest test/
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)

configure_file(../test-data/small.sgy write.sgy COPYONLY)

add_test(
    NAME python.example.about
    COMMAND ${Python_EXECUTABLE} -m examples.about
                ../test-data/small.sgy INLINE_3D CROSSLINE_3D
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)
add_test(
    NAME python.example.write
    COMMAND ${Python_EXECUTABLE} -m examples.write
                ${CMAKE_CURRENT_BINARY_DIR}/write.sgy
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)
add_test(
    NAME python.example.makefile
    COMMAND ${Python_EXECUTABLE} -m examples.make-file
                ../test-data/large-file.sgy 20 1 20 1 20
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)
add_test(
    NAME python.example.makepsfile
    COMMAND ${Python_EXECUTABLE} -m examples.make-ps-file
                ../test-data/small-prestack.sgy 10 1 5 1 4 1 3
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)
add_test(
    NAME python.example.subcube
    COMMAND ${Python_EXECUTABLE} -m examples.copy-sub-cube
                ../test-data/small.sgy ${CMAKE_CURRENT_BINARY_DIR}/copy.sgy
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)
add_test(
    NAME python.example.rotate
    COMMAND ${Python_EXECUTABLE} -m examples.make-rotated-copies
                ../test-data/small.sgy ex-rotate.sgy ${CMAKE_CURRENT_BINARY_DIR}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)
add_test(
    NAME python.example.scan_min_max
    COMMAND ${Python_EXECUTABLE} -m examples.scan_min_max
                ../test-data/small.sgy
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)
add_test(
    NAME python.example.multi-text
    COMMAND ${Python_EXECUTABLE} -m examples.make-multiple-text
                test-artifacts/multi-text.sgy
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)
