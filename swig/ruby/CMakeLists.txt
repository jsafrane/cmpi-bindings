#
# CMakeLists.txt for cmpi-bindings/swig/ruby
#

enable_testing()
ADD_SUBDIRECTORY(tests)

EXECUTE_PROCESS(COMMAND ${RUBY_EXECUTABLE} -r rbconfig -e "print RbConfig::CONFIG['ruby_version']" OUTPUT_VARIABLE RUBY_VERSION)

EXECUTE_PROCESS(COMMAND ${RUBY_EXECUTABLE} -r rbconfig -e "print RbConfig::CONFIG['vendorarchdir']" OUTPUT_VARIABLE RUBY_ARCH_DIR)
EXECUTE_PROCESS(COMMAND ${RUBY_EXECUTABLE} -r rbconfig -e "print RbConfig::CONFIG['vendorlibdir']" OUTPUT_VARIABLE RUBY_LIB_DIR)

IF( "${RUBY_ARCH_DIR}" STREQUAL "nil" OR "${RUBY_LIB_DIR}" STREQUAL "nil")
  MESSAGE(STATUS "Ruby too old, no vendor_dir, going for site_dir")
  EXECUTE_PROCESS(COMMAND ${RUBY_EXECUTABLE} -r rbconfig -e "print RbConfig::CONFIG['sitearchdir']" OUTPUT_VARIABLE RUBY_ARCH_DIR)
  EXECUTE_PROCESS(COMMAND ${RUBY_EXECUTABLE} -r rbconfig -e "print RbConfig::CONFIG['sitelibdir']" OUTPUT_VARIABLE RUBY_LIB_DIR)
ENDIF( "${RUBY_ARCH_DIR}" STREQUAL "nil" OR "${RUBY_LIB_DIR}" STREQUAL "nil")

MESSAGE(STATUS "Ruby executable: ${RUBY_EXECUTABLE}")
MESSAGE(STATUS "Ruby version: ${RUBY_VERSION}")
MESSAGE(STATUS "Ruby library: ${RUBY_LIBRARY}")
MESSAGE(STATUS "Ruby arch: ${RUBY_ARCH}")
MESSAGE(STATUS "Ruby arch dir: ${RUBY_ARCH_DIR}")
MESSAGE(STATUS "Ruby lib dir: ${RUBY_LIB_DIR}")
MESSAGE(STATUS "Ruby include path: ${RUBY_INCLUDE_PATH}")

SET( SWIG_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/cmpi_wrap.c" )
SET( SWIG_INPUT "${CMAKE_CURRENT_SOURCE_DIR}/../cmpi.i" )

ADD_CUSTOM_COMMAND (
   OUTPUT  ${SWIG_OUTPUT}
   COMMAND ${CMAKE_COMMAND} -E echo_append "Creating wrapper code for ruby..."
   COMMAND ${SWIG_EXECUTABLE} -ruby -o ${SWIG_OUTPUT} -outdir ${CMAKE_CURRENT_BINARY_DIR} -DHAVE_CMPI_BROKER=1 -I/usr/include ${SWIG_INPUT}
   COMMAND ${CMAKE_COMMAND} -E echo "Done."
   WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
   DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/../*.i
   DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/../../src/*.c
)


SET( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC" )

INCLUDE_DIRECTORIES( ${CMAKE_CURRENT_SOURCE_DIR}/.. )
INCLUDE_DIRECTORIES( ${CMAKE_SOURCE_DIR}/include/cmpi )
INCLUDE_DIRECTORIES( ${RUBY_INCLUDE_PATH} )
INCLUDE_DIRECTORIES( ${RUBY_INCLUDE_PATH}/${RUBY_ARCH} )
INCLUDE_DIRECTORIES( ${CMPI_INCLUDE_DIR} )

ADD_DEFINITIONS(-DCMPI_PLATFORM_LINUX_GENERIC_GNU -DCMPI_VERSION=200)
ADD_DEFINITIONS(-DTARGET_RUBY)

#
# rbCmpiProvider: provider .so
#

SET( NAME rbCmpiProvider )
ADD_LIBRARY( ${NAME} SHARED ${SWIG_OUTPUT})
TARGET_LINK_LIBRARIES( ${NAME} ${RUBY_LIBRARY} )

INSTALL(TARGETS ${NAME} LIBRARY DESTINATION ${CMPI_LIBRARY_DIR})

#
# cmpi: provider implementation
#
INSTALL(FILES cmpi.rb DESTINATION ${RUBY_LIB_DIR})
# create (empty) cmpi/providers (because git doesn't care)
FILE(MAKE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/cmpi/providers)
# copy cmpi/*.rb to LIB_DIR
INSTALL(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/cmpi DESTINATION ${RUBY_LIB_DIR}
        PATTERN ".rb"
	PATTERN "*~" EXCLUDE)

# rdoc
IF(RUBY_VERSION VERSION_LESS "1.9.1")
SET(rdoc_dir "${CMAKE_CURRENT_BINARY_DIR}/html")
ADD_CUSTOM_COMMAND (
   OUTPUT ${rdoc_dir}
   COMMAND ${CMAKE_COMMAND} -E echo_append "Creating rdoc documentation ..."
   COMMAND rm -rf ${rdoc_dir}
   COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/rdoc -o ${rdoc_dir} cmpi.i *.i ruby/cmpi.rb ruby/cmpi/*.rb
   COMMAND ${CMAKE_COMMAND} -E echo "Done."
   WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/swig
   DEPENDS ${CMAKE_SOURCE_DIR}/swig/*.i ${CMAKE_SOURCE_DIR}/swig/ruby/*.rb ${CMAKE_SOURCE_DIR}/swig/ruby/cmpi/*.rb
)
ADD_CUSTOM_TARGET(ruby_rdoc ALL DEPENDS "${rdoc_dir}")
ADD_DEPENDENCIES(ruby_rdoc ${NAME})
ENDIF(RUBY_VERSION VERSION_LESS "1.9.1")

IF( BUILD_RUBY_GEM )
#
# Ruby GEM
#

MESSAGE(STATUS "Building Ruby GEM")

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmpi-bindings.gemspec.in ${CMAKE_CURRENT_BINARY_DIR}/cmpi-bindings.gemspec)

SET(GEM_NAME "cmpi-bindings-${VERSION}.gem")
ADD_CUSTOM_TARGET(ruby_gem ALL DEPENDS ${GEM_NAME})
ADD_DEPENDENCIES(ruby_gem ${SWIG_OUTPUT})

ADD_CUSTOM_COMMAND (
   OUTPUT  ${GEM_NAME}
   WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
   COMMAND "gem" "build" "${CMAKE_CURRENT_BINARY_DIR}/cmpi-bindings.gemspec"
   DEPENDS "ext"
   DEPENDS ${rdoc_dir}
   DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/cmpi-bindings.gemspec
)

#
# Prep GEM files
# gemspec needs _relative_ pathes
# so create ext and lib here and copy files
#
ADD_CUSTOM_COMMAND (
  OUTPUT "ext"
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  COMMAND "rm" "-rf" "lib"
  COMMAND "rm" "-rf" "ext"
  COMMAND "mkdir" "-p" "ext/cmpi-bindings"
  COMMAND "cp" "${CMAKE_CURRENT_SOURCE_DIR}/extconf.rb" "ext/cmpi-bindings"
  COMMAND "cp" "${CMAKE_SOURCE_DIR}/swig/*.h" "ext/cmpi-bindings"
  COMMAND "cp" "${CMAKE_CURRENT_BINARY_DIR}/*.c" "ext/cmpi-bindings"
  COMMAND "mkdir" "-p" "ext/src"
  COMMAND "cp" "${CMAKE_SOURCE_DIR}/src/cmpi_provider.c" "ext/src"
  COMMAND "cp" "${CMAKE_SOURCE_DIR}/src/target_ruby.c" "ext/src"
  COMMAND "mkdir" "-p" "lib"
  COMMAND "cp" "${CMAKE_CURRENT_SOURCE_DIR}/cmpi.rb" "lib"
  COMMAND "cp" "-a" "${CMAKE_CURRENT_SOURCE_DIR}/cmpi" "lib"

  DEPENDS ${CMAKE_SOURCE_DIR}/src/cmpi_provider.c
  DEPENDS ${CMAKE_SOURCE_DIR}/src/target_ruby.c
  DEPENDS ${CMAKE_SOURCE_DIR}/swig/*.h
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/extconf.rb
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/cmpi.rb
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/cmpi/*.rb
  DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/*.c
)

ENDIF( BUILD_RUBY_GEM )

#
# Leave this to %doc in the .spec file
#INSTALL(DIRECTORY "${rdoc_dir}" DESTINATION ${DOC_INSTALL_DIR})
