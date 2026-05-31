# libs should be static
set(BUILD_SHARED_LIBS ON)

# use PIC code for link into shared lib
if (UNIX)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
endif()

include_directories(./vl)

set(FEATS
    vl/covdet.c
    vl/generic.c
    vl/host.c
    vl/imopv.c
    vl/imopv_sse2.c
    vl/mathop.c
    vl/mathop_avx.c
    vl/mathop_sse2.c
    vl/mser.c
    vl/random.c
    vl/scalespace.c
    vl/sift.c
    vl/stringop.c
)

set(FEATS_H
    vl/covdet.h
    vl/float.th
    vl/generic.h
    vl/host.h
    vl/imopv.h
    vl/imopv_sse2.h
    vl/mathop.h
    vl/mathop_avx.h
    vl/mathop_sse2.h
    vl/mser.h
    vl/random.h
    vl/scalespace.h
    vl/sift.h
    vl/stringop.h
)

set_source_files_properties(${FEATS} ${FEATS_H} PROPERTIES LANGUAGE C)
set_source_files_properties(${FEATS_H} PROPERTIES HEADER_FILE_ONLY TRUE)

set(SIMD_DEFINITIONS)
if (NOT ALICEVISION_HAVE_AVX)
    list(APPEND SIMD_DEFINITIONS "-DVL_DISABLE_AVX")
endif()
if (NOT ALICEVISION_HAVE_SSE)
    list(APPEND SIMD_DEFINITIONS "-DVL_DISABLE_SSE2")
endif()

# The bundled VLFeat AVX kernels (vl/mathop_avx.c) use AVX intrinsics that, on
# GCC/Clang, require -mavx to compile (MSVC gets /arch:AVX globally). Without it the
# file fails with "#error Compiling AVX functions but AVX does not seem to be supported"
# and a cascade of unknown-type errors. Scope the flag to that single translation unit
# so the rest of the library keeps its default ISA baseline.
if (ALICEVISION_HAVE_AVX AND (CMAKE_C_COMPILER_ID MATCHES "GNU" OR CMAKE_C_COMPILER_ID MATCHES "Clang"))
    set_source_files_properties(vl/mathop_avx.c PROPERTIES COMPILE_OPTIONS "-mavx")
endif()

alicevision_add_library(vlsift
    SOURCES
        ${FEATS} ${FEATS_H}
    PRIVATE_LINKS
        ${ALICEVISION_OPENMP_C_TARGETS}
    PUBLIC_DEFINITIONS
        ${SIMD_DEFINITIONS}
    PRIVATE_DEFINITIONS
        -DVL_BUILD_DLL
)

set_property(TARGET vlsift
    PROPERTY FOLDER nonFree
)

install(
    DIRECTORY .
    DESTINATION include/aliceVision_dependencies/nonFree/sift
    COMPONENT headers
    FILES_MATCHING PATTERN "*.hpp" PATTERN "*.h"
)
