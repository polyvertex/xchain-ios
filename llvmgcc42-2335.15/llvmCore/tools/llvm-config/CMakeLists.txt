include(TestBigEndian)

include(FindPerl)
if( NOT PERL_FOUND )
  message(FATAL_ERROR "Perl required but not found!")
endif( NOT PERL_FOUND )

set(PERL ${PERL_EXECUTABLE})
set(VERSION PACKAGE_VERSION)
set(PREFIX ${LLVM_BINARY_DIR}) # TODO: Root for `make install'.
set(abs_top_srcdir ${LLVM_MAIN_SRC_DIR})
set(abs_top_builddir ${LLVM_BINARY_DIR})
execute_process(COMMAND date
  OUTPUT_VARIABLE LLVM_CONFIGTIME
  OUTPUT_STRIP_TRAILING_WHITESPACE)
# LLVM_ON_UNIX and LLVM_ON_WIN32 already set.
# those are set to blank by `autoconf' on MinGW, so it seems they are not required:
#set(LLVMGCCDIR "")
#set(LLVMGCC "")
#set(LLVMGXX "")
test_big_endian(IS_BIG_ENDIAN)
if( IS_BIG_ENDIAN )
  set(ENDIAN "big")
else( IS_BIG_ENDIAN )
  set(ENDIAN "little")
endif( IS_BIG_ENDIAN )
set(SHLIBEXT ${LTDL_SHLIB_EXT})
#EXEEXT already set.
set(OS "${CMAKE_SYSTEM}")
set(ARCH "${LLVM_NATIVE_ARCH}")

get_system_libs(LLVM_SYSTEM_LIBS_LIST)
foreach(l ${LLVM_SYSTEM_LIBS_LIST})
  set(LLVM_SYSTEM_LIBS ${LLVM_SYSTEM_LIBS} "-l${l}")
endforeach()

foreach(c ${LLVM_TARGETS_TO_BUILD})
  set(TARGETS_BUILT "${TARGETS_BUILT} ${c}")
endforeach(c)
set(TARGETS_TO_BUILD ${TARGETS_BUILT})
set(TARGET_HAS_JIT "1")  # TODO

# Avoids replacement at config-time:
set(LLVM_CPPFLAGS "@LLVM_CPPFLAGS@")
set(LLVM_CFLAGS "@LLVM_CFLAGS@")
set(LLVM_CXXFLAGS "@LLVM_CXXFLAGS@")
set(LLVM_LDFLAGS "@LLVM_LDFLAGS@")
set(LIBS "@LIBS@")
set(LLVM_BUILDMODE "@LLVM_BUILDMODE@")

configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/llvm-config.in.in
  ${CMAKE_CURRENT_BINARY_DIR}/llvm-config.in
  @ONLY
)

set(LIBDEPS ${CMAKE_CURRENT_BINARY_DIR}/LibDeps.txt)
set(LIBDEPS_TMP ${CMAKE_CURRENT_BINARY_DIR}/LibDeps.txt.tmp)
set(FINAL_LIBDEPS ${CMAKE_CURRENT_BINARY_DIR}/FinalLibDeps.txt)
set(LLVM_CONFIG ${LLVM_TOOLS_BINARY_DIR}/llvm-config)
set(LLVM_CONFIG_IN ${CMAKE_CURRENT_BINARY_DIR}/llvm-config.in)

if( CMAKE_CROSSCOMPILING )
  set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY)
endif()

find_program(NM_PATH nm PATH_SUFFIXES /bin)

if( NOT NM_PATH )
  message(FATAL_ERROR "`nm' not found")
endif()

add_custom_command(OUTPUT ${LIBDEPS_TMP}
  COMMAND ${PERL_EXECUTABLE} ${LLVM_MAIN_SRC_DIR}/utils/GenLibDeps.pl -flat ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}/${CMAKE_CFG_INTDIR} ${NM_PATH} > ${LIBDEPS_TMP}
  DEPENDS ${llvm_libs}
  COMMENT "Regenerating ${LIBDEPS_TMP}")

add_custom_command(OUTPUT ${LIBDEPS}
  COMMAND ${CMAKE_COMMAND} -E copy_if_different ${LIBDEPS_TMP} ${LIBDEPS}
  DEPENDS ${LIBDEPS_TMP}
  COMMENT "Updated ${LIBDEPS} because dependencies changed")

add_custom_command(OUTPUT ${FINAL_LIBDEPS}
  COMMAND ${PERL_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/find-cycles.pl < ${LIBDEPS} > ${FINAL_LIBDEPS} || ${CMAKE_COMMAND} -E remove -f ${FINAL_LIBDEPS}
  DEPENDS ${LIBDEPS}
  COMMENT "Checking for cyclic dependencies between LLVM libraries.")

set(C_FLGS "${CMAKE_C_FLAGS_${uppercase_CMAKE_BUILD_TYPE}} ${LLVM_DEFINITIONS}")
set(CXX_FLGS "${CMAKE_CXX_FLAGS_${uppercase_CMAKE_BUILD_TYPE}} ${LLVM_DEFINITIONS}")
set(CPP_FLGS "${CMAKE_CPP_FLAGS_${uppercase_CMAKE_BUILD_TYPE}} ${LLVM_DEFINITIONS}")

add_custom_command(OUTPUT ${LLVM_CONFIG}
  COMMAND echo 's!@LLVM_CPPFLAGS@!${CPP_FLGS}!' > temp.sed
  COMMAND echo 's!@LLVM_CFLAGS@!${C_FLGS}!' >> temp.sed
  COMMAND echo 's!@LLVM_CXXFLAGS@!${CXX_FLGS}!' >> temp.sed
  # TODO: Use general flags for linking! not just for shared libs:
  COMMAND echo 's!@LLVM_LDFLAGS@!${CMAKE_SHARED_LINKER_FLAGS}!' >> temp.sed
  COMMAND echo 's!@LIBS@!${LLVM_SYSTEM_LIBS}!' >> temp.sed
  COMMAND echo 's!@LLVM_BUILDMODE@!${CMAKE_BUILD_TYPE}!' >> temp.sed
  COMMAND sed -f temp.sed < ${LLVM_CONFIG_IN} > ${LLVM_CONFIG}
  COMMAND ${CMAKE_COMMAND} -E remove -f temp.sed
  COMMAND cat ${FINAL_LIBDEPS} >> ${LLVM_CONFIG}
  COMMAND chmod +x ${LLVM_CONFIG}
  DEPENDS ${FINAL_LIBDEPS} ${LLVM_CONFIG_IN}
  COMMENT "Building llvm-config script."
  )

add_custom_target(llvm-config.target ALL
  DEPENDS ${LLVM_CONFIG})

add_dependencies(llvm-config.target ${llvm_lib_targets})

# Make sure that llvm-config builds before the llvm tools, so we have
# LibDeps.txt and can use it for updating the hard-coded library
# dependencies on cmake/modules/LLVMLibDeps.cmake when the tools'
# build fail due to outdated dependencies:
set(LLVM_COMMON_DEPENDS ${LLVM_COMMON_DEPENDS} llvm-config.target)

install(FILES ${LLVM_CONFIG}
  PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE
  WORLD_READ WORLD_EXECUTE
  DESTINATION bin)


# Regeneration of library dependencies.

# See the comments at the end of cmake/modules/LLVMConfig.cmake for
# notes and guidelines.

set(LLVMLibDeps ${LLVM_MAIN_SRC_DIR}/cmake/modules/LLVMLibDeps.cmake)
set(LLVMLibDeps_TMP ${CMAKE_CURRENT_BINARY_DIR}/LLVMLibDeps.cmake.tmp)

add_custom_command(OUTPUT ${LLVMLibDeps_TMP}
  COMMAND sed -e s'@\\.a@@g' -e s'@\\.so@@g' -e 's@libLLVM@LLVM@g' -e 's@: @ @' -e 's@\\\(.*\\\)@set\(MSVC_LIB_DEPS_\\1\)@' ${FINAL_LIBDEPS} > ${LLVMLibDeps_TMP}
  COMMAND ${CMAKE_COMMAND} -E copy_if_different ${LLVMLibDeps_TMP} ${LLVMLibDeps}
  DEPENDS ${FINAL_LIBDEPS}
  COMMENT "Updating cmake library dependencies file ${LLVMLibDeps}"
  )

if( LLVM_TARGETS_TO_BUILD STREQUAL LLVM_ALL_TARGETS )
  add_custom_target(llvmlibdeps.target ALL DEPENDS ${LLVMLibDeps_TMP})
  add_dependencies(llvmlibdeps.target llvm-config.target)
  set(LLVM_COMMON_DEPENDS ${LLVM_COMMON_DEPENDS} llvmlibdeps.target)
endif()

set(LLVM_COMMON_DEPENDS ${LLVM_COMMON_DEPENDS} PARENT_SCOPE)
