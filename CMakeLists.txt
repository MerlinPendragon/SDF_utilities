project(SDF_UTILITIES)
cmake_minimum_required(VERSION 2.8.11)

set(CFLAGS "${CMAKE_C_FLAGS} -g")
set(CXXFLAGS "${CMAKE_CXX_FLAGS} -g")
if(${CMAKE_BUILD_TYPE}x STREQUAL "Debugx")
   add_definitions(-DSDF_DEBUG_ALL)
   set(DEBUG_OPT --debug)
   set(OPT_LEVEL 0)
   if(${CMAKE_C_COMPILER_ID} STREQUAL "GNU")
       set(EXTRA_COMPILE_ARGS -gdwarf-2)
   endif()
   set(CFLAGS "${CFLAGS} ${CMAKE_C_FLAGS_DEBUG}")
   set(CXXFLAGS "${CXXFLAGS} ${CMAKE_CXX_FLAGS_DEBUG}")
else()
   set(OPT_LEVEL 3)
endif()
add_definitions(-D_XOPEN_SOURCE=600)

include_directories(${SDFC_INCLUDE_DIR} ${CMAKE_CURRENT_BINARY_DIR})

add_custom_target(
    commit_info.h
    COMMAND ${CMAKE_COMMAND} -DSRC_DIR:PATH=${CMAKE_CURRENT_SOURCE_DIR} -P
        ${CMAKE_CURRENT_SOURCE_DIR}/gen_commit_string.cmake)

add_executable(sdf2ascii sdf2ascii.c)
add_dependencies(sdf2ascii commit_info.h)
target_link_libraries(sdf2ascii sdfc dl m)

add_executable(sdffilter sdffilter.c sdf_vtk_writer.c)
add_dependencies(sdffilter commit_info.h)
target_link_libraries(sdffilter sdfc dl m)

add_executable(sdfdiff sdfdiff.c)
add_dependencies(sdfdiff commit_info.h)
target_link_libraries(sdfdiff sdfc dl m)

if(PARALLEL)
    add_definitions(-DPARALLEL)
    include_directories(${MPI_C_INCLUDE_PATH})
    target_link_libraries(sdf2ascii ${MPI_C_LIBRARIES})
    target_link_libraries(sdffilter ${MPI_C_LIBRARIES})
    target_link_libraries(sdfdiff ${MPI_C_LIBRARIES})
endif()

add_custom_target(sdf-utilities DEPENDS sdf2ascii sdffilter sdfdiff)
install(TARGETS sdf2ascii sdffilter sdfdiff DESTINATION bin COMPONENT
        sdf-utilities)

set(BUILD_SDFPY ON CACHE BOOL "Build the Python interface")
if(BUILD_SDFPY)
   find_package(PythonInterp)
   configure_file(setup.py.in setup.py)
   configure_file(setup.cfg.in setup.cfg)
   add_custom_command(OUTPUT build/timestamp
      COMMAND ${PYTHON_EXECUTABLE} setup.py build ${DEBUG_OPT}
      COMMAND ${CMAKE_COMMAND} -E touch build/timestamp
      DEPENDS setup.py sdf_python.c sdfc commit_info.h)
   add_custom_target(sdfpy ALL DEPENDS build/timestamp)

   configure_file(sdfpy_install.cmake.in sdfpy_install.cmake)
   install(SCRIPT ${PROJECT_BINARY_DIR}/sdfpy_install.cmake COMPONENT sdfpy)
endif()
