cmake_minimum_required(VERSION 3.15)

if(POLICY CMP0091)
  cmake_policy(SET CMP0091 NEW)
endif()

project(libvpx VERSION 1.15.0 LANGUAGES C CXX)

option(CONFIG_VP8 "Enable VP8 codec" ON)
option(CONFIG_VP9 "Enable VP9 codec" ON)
option(CONFIG_ENCODERS "Enable encoders" ON)
option(CONFIG_DECODERS "Enable decoders" ON)
option(CONFIG_MULTITHREAD "Enable multithread support" ON)
option(CONFIG_STATIC "Build static library" ON)
option(CONFIG_SHARED "Build shared library" OFF)
option(CONFIG_VP9_HIGHBITDEPTH "Enable VP9 high bit depth" ON)
option(CONFIG_POSTPROC "Enable postprocessing" ON)
option(CONFIG_SPATIAL_RESAMPLING "Enable spatial resampling" ON)
option(CONFIG_TEMPORAL_DENOISING "Enable temporal denoising" ON)
option(CONFIG_INTERNAL_STATS "Enable internal stats" OFF)
option(CONFIG_ERROR_CONCEALMENT "Enable error concealment" OFF)
option(CONFIG_SIMD "Enable SIMD optimizations" ON)
option(CONFIG_ASM_NASM "Enable NASM x86 assembly optimizations" ON)
option(CONFIG_RUNTIME_CPU_DETECT "Enable runtime CPU detection" ON)
option(CONFIG_REALTIME_ONLY "Enable realtime only" OFF)
option(CONFIG_ONTHEFLY_BITPACKING "Enable on-the-fly bitpacking" OFF)
option(CONFIG_EXAMPLES "Build example executables" OFF)

set(VPX_ARCH_X86 OFF)
set(VPX_ARCH_X86_64 OFF)
set(VPX_ARCH_ARM OFF)
set(VPX_ARCH_AARCH64 OFF)

set(VPX_TARGET_PROCESSOR
  "${CMAKE_SYSTEM_PROCESSOR};${CMAKE_VS_PLATFORM_NAME};${CMAKE_GENERATOR_PLATFORM};${CMAKE_C_COMPILER_ARCHITECTURE_ID}"
)

if(VPX_TARGET_PROCESSOR MATCHES "aarch64|arm64|ARM64|ARM64EC")
  set(VPX_ARCH_AARCH64 ON)
elseif(VPX_TARGET_PROCESSOR MATCHES "x86_64|AMD64|amd64|x64|X64")
  set(VPX_ARCH_X86_64 ON)
elseif(VPX_TARGET_PROCESSOR MATCHES "i[3-6]86|x86")
  set(VPX_ARCH_X86 ON)
elseif(VPX_TARGET_PROCESSOR MATCHES "arm|ARM")
  set(VPX_ARCH_ARM ON)
endif()

# Enable only the assembler language needed by the active architecture. NASM is
# optional so x86/x86_64 builds can be configured without assembly SIMD sources.
include(CheckLanguage)
if((VPX_ARCH_X86 OR VPX_ARCH_X86_64) AND CONFIG_SIMD AND CONFIG_ASM_NASM)
  set(CMAKE_ASM_NASM_MSVC_RUNTIME_LIBRARY "")
  set(CMAKE_ASM_NASM_SOURCE_FILE_EXTENSIONS "asm")
  if(WIN32)
    if(VPX_ARCH_X86_64)
      set(CMAKE_ASM_NASM_OBJECT_FORMAT "win64")
    elseif(VPX_ARCH_X86)
      set(CMAKE_ASM_NASM_OBJECT_FORMAT "win32")
    endif()
  elseif(UNIX)
    if(VPX_ARCH_X86_64)
      set(CMAKE_ASM_NASM_OBJECT_FORMAT "elf64")
    elseif(VPX_ARCH_X86)
      set(CMAKE_ASM_NASM_OBJECT_FORMAT "elf32")
    endif()
  endif()

  check_language(ASM_NASM)
  if(CMAKE_ASM_NASM_COMPILER)
    enable_language(ASM_NASM)
    set(HAVE_X86_ASM ON)
    # Add include path for NASM to find vpx_config.asm
    set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -I${CMAKE_CURRENT_BINARY_DIR}/ -w-reloc-rel-dword")
  endif()
elseif(VPX_ARCH_ARM OR VPX_ARCH_AARCH64)
  enable_language(ASM)
endif()

# ============================================================================
# SIMD feature detection
# ============================================================================
include(CheckCSourceCompiles)

# SIMD flags must be assigned before feature checks so Clang/GCC probe features
# with the same per-source flags used for the actual optimized files.
if(MSVC)
  set(VPX_MMX_FLAGS "")
  set(VPX_SSE2_FLAGS "")
  set(VPX_SSE3_FLAGS "")
  set(VPX_SSSE3_FLAGS "")
  set(VPX_SSE4_1_FLAGS "")
  set(VPX_AVX_FLAGS "/arch:AVX")
  set(VPX_AVX2_FLAGS "/arch:AVX2")
  set(VPX_AVX512_FLAGS "/arch:AVX512")
else()
  set(VPX_MMX_FLAGS "-mmmx")
  set(VPX_SSE2_FLAGS "-msse2")
  set(VPX_SSE3_FLAGS "-msse3")
  set(VPX_SSSE3_FLAGS "-mssse3")
  set(VPX_SSE4_1_FLAGS "-msse4.1")
  set(VPX_AVX_FLAGS "-mavx")
  set(VPX_AVX2_FLAGS "-mavx2")
  set(VPX_AVX512_FLAGS "-mavx512f -mavx512cd -mavx512bw -mavx512dq -mavx512vl")
  if(VPX_ARCH_AARCH64)
    set(VPX_NEON_FLAGS "")
  else()
    set(VPX_NEON_FLAGS "-mfpu=neon")
  endif()
  set(VPX_SVE_FLAGS "-march=armv8-a+sve")
  set(VPX_SVE2_FLAGS "-march=armv8-a+sve2")
endif()

# Helper function to check for SIMD support with proper flags
macro(vpx_check_simd feature_name code flags_var)
  set(SAFE_CMAKE_REQUIRED_FLAGS "${CMAKE_REQUIRED_FLAGS}")
  if(NOT MSVC)
    set(CMAKE_REQUIRED_FLAGS "${${flags_var}}")
  endif()
  check_c_source_compiles("${code}" HAVE_${feature_name})
  set(CMAKE_REQUIRED_FLAGS "${SAFE_CMAKE_REQUIRED_FLAGS}")
endmacro()

# X86 SIMD detection
if(CONFIG_SIMD AND (VPX_ARCH_X86 OR VPX_ARCH_X86_64))
  # Check for MMX
  vpx_check_simd(MMX "
    #include <mmintrin.h>
    int main() { _m_from_int(0); return 0; }
  " VPX_MMX_FLAGS)

  # Check for SSE2
  vpx_check_simd(SSE2 "
    #include <emmintrin.h>
    int main() { __m128i a = _mm_setzero_si128(); return 0; }
  " VPX_SSE2_FLAGS)

  # Check for SSE3
  vpx_check_simd(SSE3 "
    #include <pmmintrin.h>
    int main() { __m128 a = _mm_setzero_ps(); return 0; }
  " VPX_SSE3_FLAGS)

  # Check for SSSE3
  vpx_check_simd(SSSE3 "
    #include <tmmintrin.h>
    int main() { __m128i a = _mm_abs_epi8(_mm_setzero_si128()); return 0; }
  " VPX_SSSE3_FLAGS)

  # Check for SSE4.1
  vpx_check_simd(SSE4_1 "
    #include <smmintrin.h>
    int main() { int a = _mm_extract_epi32(_mm_setzero_si128(), 0); return 0; }
  " VPX_SSE4_1_FLAGS)

  # Check for AVX
  vpx_check_simd(AVX "
    #include <immintrin.h>
    int main() { __m256 a = _mm256_setzero_ps(); return 0; }
  " VPX_AVX_FLAGS)

  # Check for AVX2
  vpx_check_simd(AVX2 "
    #include <immintrin.h>
    int main() { __m256i a = _mm256_setzero_si256(); return 0; }
  " VPX_AVX2_FLAGS)

  # Check for AVX512
  vpx_check_simd(AVX512 "
    #include <immintrin.h>
    int main() { __m512i a = _mm512_setzero_si512(); return 0; }
  " VPX_AVX512_FLAGS)
else()
  foreach(x86_feature MMX SSE2 SSE3 SSSE3 SSE4_1 AVX AVX2 AVX512)
    set(HAVE_${x86_feature} OFF CACHE INTERNAL "" FORCE)
  endforeach()
  set(HAVE_X86_ASM OFF CACHE INTERNAL "" FORCE)
endif()

# ARM NEON detection
if(CONFIG_SIMD AND (VPX_ARCH_ARM OR VPX_ARCH_AARCH64))
  if(VPX_ARCH_AARCH64)
    # AArch64 always has NEON
    set(HAVE_NEON ON)

    # Check for NEON dotprod
    vpx_check_simd(NEON_DOTPROD "
      #include <arm_neon.h>
      int main() {
        uint32x4_t a, b;
        uint32x4_t r = vdotq_u32(a, b, b);
        return 0;
      }
    " VPX_NEON_FLAGS)

    # Check for NEON i8mm
    vpx_check_simd(NEON_I8MM "
      #include <arm_neon.h>
      int main() {
        int8x16_t a, b;
        int32x4_t r = vmmlaq_s32(vdupq_n_s32(0), a, b);
        return 0;
      }
    " VPX_NEON_FLAGS)

    vpx_check_simd(SVE "
      #include <arm_sve.h>
      #include <arm_neon_sve_bridge.h>
      int main() {
        svint32_t a;
        return 0;
      }
    " VPX_SVE_FLAGS)

    vpx_check_simd(SVE2 "
      #include <arm_sve.h>
      #include <arm_neon_sve_bridge.h>
      int main() {
        svint32_t a;
        return 0;
      }
    " VPX_SVE2_FLAGS)
  else()
    # Check for NEON on ARM32
    vpx_check_simd(NEON "
      #include <arm_neon.h>
      int main() {
        int32x4_t a = vdupq_n_s32(0);
        return 0;
      }
    " VPX_NEON_FLAGS)
  endif()

  # ARM32 NEON .asm sources use assembler syntax that this CMake build does not
  # currently compile, so keep C NEON implementations enabled.
  set(HAVE_NEON_ASM OFF)
else()
  foreach(arm_feature NEON NEON_DOTPROD NEON_I8MM NEON_ASM SVE SVE2)
    set(HAVE_${arm_feature} OFF CACHE INTERNAL "" FORCE)
  endforeach()
endif()

# ============================================================================
# Compiler flags
# ============================================================================
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Common flags
if(MSVC)
  add_compile_options(
    $<$<COMPILE_LANGUAGE:C>:/W3>
    $<$<COMPILE_LANGUAGE:CXX>:/W3>
  )
else()
  add_compile_options(
    $<$<COMPILE_LANGUAGE:C>:-Wall>
    $<$<COMPILE_LANGUAGE:C>:-Wextra>
    $<$<COMPILE_LANGUAGE:CXX>:-Wall>
    $<$<COMPILE_LANGUAGE:CXX>:-Wextra>
  )
endif()

# Helper function to add SIMD sources with proper flags
function(vpx_add_simd_source target_list simd_type)
  set(files ${ARGN})
  set(enabled_files "")
  foreach(file ${files})
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${file}")
      if(file MATCHES "\\.asm$")
        if(HAVE_X86_ASM)
          list(APPEND enabled_files ${file})
        endif()
      else()
        list(APPEND enabled_files ${file})
      endif()
    endif()
  endforeach()

  list(APPEND ${target_list} ${enabled_files})
  set(${target_list} ${${target_list}} PARENT_SCOPE)

  set(flags "")
  if(simd_type STREQUAL "MMX")
    set(flags ${VPX_MMX_FLAGS})
  elseif(simd_type STREQUAL "SSE2")
    set(flags ${VPX_SSE2_FLAGS})
  elseif(simd_type STREQUAL "SSE3")
    set(flags ${VPX_SSE3_FLAGS})
  elseif(simd_type STREQUAL "SSSE3")
    set(flags ${VPX_SSSE3_FLAGS})
  elseif(simd_type STREQUAL "SSE4_1")
    set(flags ${VPX_SSE4_1_FLAGS})
  elseif(simd_type STREQUAL "AVX")
    set(flags ${VPX_AVX_FLAGS})
  elseif(simd_type STREQUAL "AVX2")
    set(flags ${VPX_AVX2_FLAGS})
  elseif(simd_type STREQUAL "AVX512")
    set(flags ${VPX_AVX512_FLAGS})
  elseif(simd_type STREQUAL "NEON")
    set(flags ${VPX_NEON_FLAGS})
  elseif(simd_type STREQUAL "SVE")
    set(flags ${VPX_SVE_FLAGS})
  elseif(simd_type STREQUAL "SVE2")
    set(flags ${VPX_SVE2_FLAGS})
  endif()

  if(flags)
    foreach(file ${enabled_files})
      if(file MATCHES "\\.c$")
        set_source_files_properties(${file} PROPERTIES COMPILE_FLAGS "${flags}")
      endif()
    endforeach()
  endif()
endfunction()

# ============================================================================
# Generate configuration headers
# ============================================================================
set(CONFIG_H_IN "${CMAKE_CURRENT_SOURCE_DIR}/vpx_config.h.in")
set(CONFIG_H "${CMAKE_CURRENT_BINARY_DIR}/vpx_config.h")
set(CONFIG_ASM_IN "${CMAKE_CURRENT_SOURCE_DIR}/vpx_config.asm.in")
set(CONFIG_ASM "${CMAKE_CURRENT_BINARY_DIR}/vpx_config.asm")
set(VERSION_H_IN "${CMAKE_CURRENT_SOURCE_DIR}/vpx_version.h.in")
set(VERSION_H "${CMAKE_CURRENT_BINARY_DIR}/vpx_version.h")

# Map boolean variables to 0/1 for template substitution
set(VARS_TO_MAP
  VPX_ARCH_X86 VPX_ARCH_X86_64 VPX_ARCH_ARM VPX_ARCH_AARCH64
  CONFIG_VP8 CONFIG_VP9 CONFIG_ENCODERS CONFIG_DECODERS
  CONFIG_MULTITHREAD CONFIG_POSTPROC CONFIG_VP9_POSTPROC
  CONFIG_VP9_HIGHBITDEPTH CONFIG_SPATIAL_RESAMPLING
  CONFIG_TEMPORAL_DENOISING HAVE_MMX HAVE_SSE2 HAVE_SSE3
  HAVE_SSSE3 HAVE_SSE4_1 HAVE_AVX HAVE_AVX2 HAVE_AVX512
  HAVE_NEON HAVE_NEON_DOTPROD HAVE_NEON_I8MM HAVE_NEON_ASM
  HAVE_SVE HAVE_SVE2 HAVE_X86_ASM CONFIG_PIC
  CONFIG_RUNTIME_CPU_DETECT
  CONFIG_VP8_ENCODER CONFIG_VP8_DECODER
  CONFIG_VP9_ENCODER CONFIG_VP9_DECODER
  CONFIG_VP9_TEMPORAL_DENOISING
  CONFIG_REALTIME_ONLY CONFIG_ONTHEFLY_BITPACKING
  CONFIG_OS_SUPPORT
  CONFIG_INTERNAL_STATS
)

if(WIN32 OR UNIX)
  set(CONFIG_OS_SUPPORT ON)
endif()

if(CONFIG_VP8 AND CONFIG_ENCODERS)
  set(CONFIG_VP8_ENCODER ON)
endif()
if(CONFIG_VP8 AND CONFIG_DECODERS)
  set(CONFIG_VP8_DECODER ON)
endif()
if(CONFIG_VP9 AND CONFIG_ENCODERS)
  set(CONFIG_VP9_ENCODER ON)
endif()
if(CONFIG_VP9 AND CONFIG_DECODERS)
  set(CONFIG_VP9_DECODER ON)
endif()
if(CONFIG_VP9 AND CONFIG_TEMPORAL_DENOISING)
  set(CONFIG_VP9_TEMPORAL_DENOISING ON)
endif()

foreach(var ${VARS_TO_MAP})
  if(${var})
    set(${var}_VAL 1)
  else()
    set(${var}_VAL 0)
  endif()
endforeach()

# Find threading support
find_package(Threads QUIET)
if(Threads_FOUND AND CMAKE_USE_PTHREADS_INIT)
  set(HAVE_PTHREAD_H 1)
else()
  set(HAVE_PTHREAD_H 0)
endif()

configure_file("${CONFIG_H_IN}" "${CONFIG_H}")
configure_file("${CONFIG_ASM_IN}" "${CONFIG_ASM}")
configure_file("${VERSION_H_IN}" "${VERSION_H}")

set(MSVC_ARM_NEON_COMPAT_DIR "${CMAKE_CURRENT_BINARY_DIR}/msvc_arm_neon_compat")
if(MSVC AND (VPX_ARCH_ARM OR VPX_ARCH_AARCH64))
  file(MAKE_DIRECTORY "${MSVC_ARM_NEON_COMPAT_DIR}")
  file(WRITE "${MSVC_ARM_NEON_COMPAT_DIR}/arm_neon.h"
    "#ifndef VPX_MSVC_ARM_NEON_COMPAT_H_\n"
    "#define VPX_MSVC_ARM_NEON_COMPAT_H_\n"
    "#include <arm64_neon.h>\n"
    "#endif\n"
  )
endif()

# Generate a config file for rtcd.pl in the format it expects (KEY=yes)
set(RTCD_CONFIG_FILE "${CMAKE_CURRENT_BINARY_DIR}/vpx_config.rtcd")
set(RTCD_CONFIG_CONTENT "")
foreach(var ${VARS_TO_MAP})
  if(${var})
    string(APPEND RTCD_CONFIG_CONTENT "${var}=yes\n")
  endif()
endforeach()
file(GENERATE OUTPUT "${RTCD_CONFIG_FILE}" CONTENT "${RTCD_CONFIG_CONTENT}" NEWLINE_STYLE LF)

# Also create the headers in subdirectories to support "./header.h" relative includes
configure_file("${VERSION_H}" "${CMAKE_CURRENT_BINARY_DIR}/vpx/vpx_version.h" COPYONLY)

# ============================================================================
# RTCD (Runtime CPU Detection) header generation
# ============================================================================
set(RTCD_TARGETS "")

# Architecture detection for RTCD
if(VPX_ARCH_X86_64)
  set(RTCD_ARCH x86_64)
elseif(VPX_ARCH_X86)
  set(RTCD_ARCH x86)
elseif(VPX_ARCH_AARCH64)
  set(RTCD_ARCH arm64)
elseif(VPX_ARCH_ARM)
  set(RTCD_ARCH armv7)
endif()

set(RTCD_DISABLE_ARGS "")
foreach(rtcd_ext
    mmx sse sse2 sse3 ssse3 sse4_1 avx avx2 avx512
    neon neon_dotprod neon_i8mm sve sve2)
  string(TOUPPER "${rtcd_ext}" rtcd_ext_upper)
  if(rtcd_ext_upper STREQUAL "SSE4_1")
    set(rtcd_have_var HAVE_SSE4_1)
  elseif(rtcd_ext_upper STREQUAL "NEON_DOTPROD")
    set(rtcd_have_var HAVE_NEON_DOTPROD)
  elseif(rtcd_ext_upper STREQUAL "NEON_I8MM")
    set(rtcd_have_var HAVE_NEON_I8MM)
  else()
    set(rtcd_have_var HAVE_${rtcd_ext_upper})
  endif()

  if(NOT ${rtcd_have_var})
    list(APPEND RTCD_DISABLE_ARGS "--disable-${rtcd_ext}")
  endif()
endforeach()

find_package(Perl)
if(NOT PERL_FOUND AND CMAKE_HOST_WIN32)
  # Try to find Perl in common Git locations on Windows if not in PATH
  unset(PERL_EXECUTABLE CACHE)
  find_program(PERL_EXECUTABLE NAMES perl
    PATHS "C:/Users/Nelonn/Packages/git/usr/bin"
          "C:/Program Files/Git/usr/bin"
          "C:/Program Files (x86)/Git/usr/bin"
  )
  if(PERL_EXECUTABLE)
    set(PERL_FOUND TRUE)
  endif()
endif()

# Function to generate RTCD header
function(generate_rtcd input_pl output_h rtcd_name)
  get_filename_component(output_dir "${output_h}" DIRECTORY)
  file(MAKE_DIRECTORY "${output_dir}")
  if(PERL_FOUND AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/build/make/rtcd.pl")
    add_custom_command(
      OUTPUT "${output_h}"
      COMMAND "${PERL_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/build/make/rtcd.pl"
        --arch=${RTCD_ARCH}
        --sym=${rtcd_name}
        --config=${RTCD_CONFIG_FILE}
        ${RTCD_DISABLE_ARGS}
        "${CMAKE_CURRENT_SOURCE_DIR}/${input_pl}"
        > "${output_h}"
      DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/build/make/rtcd.pl"
              "${CMAKE_CURRENT_SOURCE_DIR}/${input_pl}"
              "${RTCD_CONFIG_FILE}"
      COMMENT "Generating ${output_h}"
      VERBATIM
    )
  else()
    # Create a minimal placeholder if rtcd.pl or perl is not available
    file(WRITE "${output_h}"
      "/* Auto-generated RTCD header - placeholder */\n"
      "#ifndef ${rtcd_name}_H\n"
      "#define ${rtcd_name}_H\n"
      "/* Runtime CPU detection requires perl and rtcd.pl */\n"
      "#endif\n"
    )
  endif()
  set(RTCD_TARGETS ${RTCD_TARGETS} "${output_h}" PARENT_SCOPE)
endfunction()

# Generate RTCD headers in both root and subdirectories to support all include styles
generate_rtcd("vpx_dsp/vpx_dsp_rtcd_defs.pl" "${CMAKE_CURRENT_BINARY_DIR}/vpx_dsp/vpx_dsp_rtcd.h" "vpx_dsp_rtcd")
generate_rtcd("vpx_scale/vpx_scale_rtcd.pl" "${CMAKE_CURRENT_BINARY_DIR}/vpx_scale/vpx_scale_rtcd.h" "vpx_scale_rtcd")
if(CONFIG_VP8)
  generate_rtcd("vp8/common/rtcd_defs.pl" "${CMAKE_CURRENT_BINARY_DIR}/vp8/common/vp8_rtcd.h" "vp8_rtcd")
endif()
if(CONFIG_VP9)
  generate_rtcd("vp9/common/vp9_rtcd_defs.pl" "${CMAKE_CURRENT_BINARY_DIR}/vp9/common/vp9_rtcd.h" "vp9_rtcd")
endif()

# ============================================================================
# Source files
# ============================================================================

# Helper function to add sources with optional architecture filtering
function(add_sources_with_arch var_prefix sources_var)
  foreach(src ${ARGN})
    list(APPEND ${sources_var} "${src}")
  endforeach()
  set(${sources_var} ${${sources_var}} PARENT_SCOPE)
endfunction()

# ============================================================================
# VPX core sources
# ============================================================================
set(VPX_SRCS
  vpx/src/vpx_codec.c
  vpx/src/vpx_image.c
  vpx/src/vpx_decoder.c
  vpx/src/vpx_encoder.c
)

set(VPX_INCLUDES
  vpx/vp8.h
  vpx/vpx_codec.h
  vpx/vpx_image.h
  vpx/vpx_frame_buffer.h
  vpx/vpx_integer.h
  vpx/internal/vpx_codec_internal.h
  vpx/internal/vpx_ratectrl_rtc.h
)

if(CONFIG_DECODERS)
  list(APPEND VPX_INCLUDES vpx/vpx_decoder.h vpx/vp8dx.h)
endif()

if(CONFIG_ENCODERS)
  list(APPEND VPX_INCLUDES vpx/vpx_encoder.h vpx/vp8cx.h vpx/vpx_tpl.h vpx/vpx_ext_ratectrl.h)
endif()

# ============================================================================
# VPX memory
# ============================================================================
set(VPX_MEM_SRCS
  vpx_mem/vpx_mem.c
)

set(VPX_MEM_INCLUDES
  vpx_mem/vpx_mem.h
  vpx_mem/include/vpx_mem_intrnl.h
)

# ============================================================================
# VPX scale
# ============================================================================
set(VPX_SCALE_SRCS
  vpx_scale/generic/yv12config.c
  vpx_scale/generic/yv12extend.c
  vpx_scale/vpx_scale_rtcd.c
)

if(CONFIG_SPATIAL_RESAMPLING)
  list(APPEND VPX_SCALE_SRCS
    vpx_scale/generic/vpx_scale.c
    vpx_scale/generic/gen_scalers.c
  )
endif()

set(VPX_SCALE_INCLUDES
  vpx_scale/yv12config.h
)

# ============================================================================
# VPX ports
# ============================================================================
set(VPX_PORTS_SRCS)

if(VPX_ARCH_X86 AND HAVE_MMX)
  vpx_add_simd_source(VPX_PORTS_SRCS MMX vpx_ports/emms_mmx.c)
endif()

if(VPX_ARCH_X86_64 AND HAVE_MMX)
  vpx_add_simd_source(VPX_PORTS_SRCS MMX vpx_ports/emms_mmx.asm)
endif()

if(WIN32 AND VPX_ARCH_X86_64 AND HAVE_X86_ASM)
  list(APPEND VPX_PORTS_SRCS vpx_ports/float_control_word.asm)
endif()

if(VPX_ARCH_AARCH64)
  list(APPEND VPX_PORTS_SRCS vpx_ports/aarch64_cpudetect.c)
elseif(VPX_ARCH_ARM)
  list(APPEND VPX_PORTS_SRCS vpx_ports/aarch32_cpudetect.c)
endif()

set(VPX_PORTS_INCLUDES
  vpx_ports/bitops.h
  vpx_ports/compiler_attributes.h
  vpx_ports/mem.h
  vpx_ports/static_assert.h
  vpx_ports/system_state.h
  vpx_ports/vpx_timer.h
  vpx_ports/emmintrin_compat.h
  vpx_ports/mem_ops.h
  vpx_ports/mem_ops_aligned.h
  vpx_ports/vpx_once.h
)

if(VPX_ARCH_X86 OR VPX_ARCH_X86_64)
  list(APPEND VPX_PORTS_INCLUDES
    vpx_ports/x86.h
  )
endif()

if(VPX_ARCH_ARM)
  list(APPEND VPX_PORTS_INCLUDES
    vpx_ports/arm_cpudetect.h
    vpx_ports/arm.h
  )
endif()

# ============================================================================
# VPX DSP
# ============================================================================
set(VPX_DSP_SRCS
  vpx_dsp/prob.c
  vpx_dsp/intrapred.c
  vpx_dsp/loopfilter.c
  vpx_dsp/skin_detection.c
  vpx_dsp/vpx_dsp_rtcd.c
)

if(CONFIG_ENCODERS)
  list(APPEND VPX_DSP_SRCS
    vpx_dsp/bitwriter.c
    vpx_dsp/bitwriter_buffer.c
    vpx_dsp/psnr.c
    vpx_dsp/sse.c
    vpx_dsp/sad.c
    vpx_dsp/subtract.c
    vpx_dsp/sum_squares.c
    vpx_dsp/variance.c
  )
endif()

if(CONFIG_DECODERS)
  list(APPEND VPX_DSP_SRCS
    vpx_dsp/bitreader.c
    vpx_dsp/bitreader_buffer.c
  )
endif()

if(CONFIG_VP9)
  list(APPEND VPX_DSP_SRCS
    vpx_dsp/vpx_convolve.c
    vpx_dsp/add_noise.c
    vpx_dsp/deblock.c
  )

  if(CONFIG_ENCODERS)
    list(APPEND VPX_DSP_SRCS
      vpx_dsp/fwd_txfm.c
      vpx_dsp/quantize.c
      vpx_dsp/avg.c
    )
  endif()

  list(APPEND VPX_DSP_SRCS
    vpx_dsp/inv_txfm.c
  )
endif()

# X86 DSP optimizations
if(VPX_ARCH_X86 OR VPX_ARCH_X86_64)
  if(HAVE_SSE2)
    vpx_add_simd_source(VPX_DSP_SRCS SSE2
      vpx_dsp/x86/post_proc_sse2.c
      vpx_dsp/x86/intrapred_sse2.asm
      vpx_dsp/x86/vpx_convolve_copy_sse2.asm
      vpx_dsp/x86/vpx_subpixel_8t_sse2.asm
      vpx_dsp/x86/vpx_subpixel_bilinear_sse2.asm
      vpx_dsp/x86/deblock_sse2.asm
      vpx_dsp/x86/add_noise_sse2.asm
      vpx_dsp/x86/sad_sse2.asm
      vpx_dsp/x86/sad4d_sse2.asm
      vpx_dsp/x86/subtract_sse2.asm
      vpx_dsp/x86/subpel_variance_sse2.asm
      vpx_dsp/x86/fwd_txfm_sse2.c
      vpx_dsp/x86/inv_txfm_sse2.c
      vpx_dsp/x86/inv_wht_sse2.asm
      vpx_dsp/x86/loopfilter_sse2.c
      vpx_dsp/x86/avg_pred_sse2.c
      vpx_dsp/x86/variance_sse2.c
      vpx_dsp/x86/sum_squares_sse2.c
      vpx_dsp/x86/avg_intrin_sse2.c
    )

    if(CONFIG_ENCODERS)
      vpx_add_simd_source(VPX_DSP_SRCS SSE2
        vpx_dsp/x86/vpx_subpixel_4t_intrin_sse2.c
        vpx_dsp/x86/quantize_sse2.c
      )
    endif()

    if(CONFIG_VP9_HIGHBITDEPTH)
      vpx_add_simd_source(VPX_DSP_SRCS SSE2
        vpx_dsp/x86/highbd_intrapred_sse2.asm
        vpx_dsp/x86/highbd_intrapred_intrin_sse2.c
        vpx_dsp/x86/vpx_high_subpixel_8t_sse2.asm
        vpx_dsp/x86/vpx_high_subpixel_bilinear_sse2.asm
        vpx_dsp/x86/highbd_loopfilter_sse2.c
        vpx_dsp/x86/highbd_inv_txfm_sse2.h
        vpx_dsp/x86/highbd_idct4x4_add_sse2.c
        vpx_dsp/x86/highbd_idct8x8_add_sse2.c
        vpx_dsp/x86/highbd_idct16x16_add_sse2.c
        vpx_dsp/x86/highbd_idct32x32_add_sse2.c
        vpx_dsp/x86/highbd_quantize_intrin_sse2.c
        vpx_dsp/x86/highbd_sad4d_sse2.asm
        vpx_dsp/x86/highbd_sad_sse2.asm
        vpx_dsp/x86/highbd_variance_sse2.c
        vpx_dsp/x86/highbd_variance_impl_sse2.asm
        vpx_dsp/x86/highbd_subpel_variance_impl_sse2.asm
      )
    endif()
  endif()

  if(HAVE_SSSE3)
    vpx_add_simd_source(VPX_DSP_SRCS SSSE3
      vpx_dsp/x86/intrapred_ssse3.asm
      vpx_dsp/x86/vpx_subpixel_8t_ssse3.asm
      vpx_dsp/x86/vpx_subpixel_bilinear_ssse3.asm
      vpx_dsp/x86/vpx_subpixel_8t_intrin_ssse3.c
      vpx_dsp/x86/inv_txfm_ssse3.c
    )
    if(CONFIG_ENCODERS)
      vpx_add_simd_source(VPX_DSP_SRCS SSSE3
        vpx_dsp/x86/quantize_ssse3.c
      )
    endif()
    if(CONFIG_VP9_HIGHBITDEPTH)
      vpx_add_simd_source(VPX_DSP_SRCS SSSE3
        vpx_dsp/x86/highbd_intrapred_intrin_ssse3.c
      )
    endif()
  endif()

  if(HAVE_SSE4_1)
    vpx_add_simd_source(VPX_DSP_SRCS SSE4_1
      vpx_dsp/x86/sse_sse4.c
    )
    if(CONFIG_VP9_HIGHBITDEPTH)
      vpx_add_simd_source(VPX_DSP_SRCS SSE4_1
        vpx_dsp/x86/highbd_idct4x4_add_sse4.c
        vpx_dsp/x86/highbd_idct8x8_add_sse4.c
        vpx_dsp/x86/highbd_idct16x16_add_sse4.c
        vpx_dsp/x86/highbd_idct32x32_add_sse4.c
      )
    endif()
  endif()

  if(HAVE_AVX)
    if(CONFIG_ENCODERS)
      vpx_add_simd_source(VPX_DSP_SRCS AVX
        vpx_dsp/x86/quantize_avx.c
      )
    endif()
  endif()

  if(HAVE_AVX2)
    vpx_add_simd_source(VPX_DSP_SRCS AVX2
      vpx_dsp/x86/vpx_subpixel_8t_intrin_avx2.c
      vpx_dsp/x86/inv_txfm_avx2.c
      vpx_dsp/x86/loopfilter_avx2.c
      vpx_dsp/x86/avg_pred_avx2.c
      vpx_dsp/x86/variance_avx2.c
      vpx_dsp/x86/sad4d_avx2.c
      vpx_dsp/x86/sad_avx2.c
      vpx_dsp/x86/subtract_avx2.c
      vpx_dsp/x86/avg_intrin_avx2.c
      vpx_dsp/x86/sse_avx2.c
    )
    if(CONFIG_ENCODERS)
      vpx_add_simd_source(VPX_DSP_SRCS AVX2
        vpx_dsp/x86/quantize_avx2.c
      )
    endif()
    if(CONFIG_VP9_HIGHBITDEPTH)
      vpx_add_simd_source(VPX_DSP_SRCS AVX2
        vpx_dsp/x86/highbd_intrapred_intrin_avx2.c
        vpx_dsp/x86/highbd_convolve_avx2.c
        vpx_dsp/x86/highbd_quantize_intrin_avx2.c
        vpx_dsp/x86/highbd_sad4d_avx2.c
        vpx_dsp/x86/highbd_sad_avx2.c
      )
    endif()
  endif()

  if(HAVE_AVX512)
    vpx_add_simd_source(VPX_DSP_SRCS AVX512
      vpx_dsp/x86/sad4d_avx512.c
      vpx_dsp/x86/sad_avx512.c
    )
    if(CONFIG_VP9_HIGHBITDEPTH)
      vpx_add_simd_source(VPX_DSP_SRCS AVX512
        vpx_dsp/x86/highbd_intrapred_intrin_avx512.c
      )
    endif()
  endif()

  if(VPX_ARCH_X86_64 AND HAVE_SSSE3)
    vpx_add_simd_source(VPX_DSP_SRCS SSSE3
      vpx_dsp/x86/fwd_txfm_ssse3_x86_64.asm
      vpx_dsp/x86/avg_ssse3_x86_64.asm
    )
  endif()

  if(VPX_ARCH_X86_64 AND HAVE_SSE2)
    vpx_add_simd_source(VPX_DSP_SRCS SSE2
      vpx_dsp/x86/ssim_opt_x86_64.asm
    )
  endif()
endif()

# ARM NEON DSP optimizations
if(HAVE_NEON)
  vpx_add_simd_source(VPX_DSP_SRCS NEON
    vpx_dsp/arm/intrapred_neon.c
    vpx_dsp/arm/deblock_neon.c
    vpx_dsp/arm/loopfilter_neon.c
    vpx_dsp/arm/idct4x4_1_add_neon.c
    vpx_dsp/arm/idct4x4_add_neon.c
    vpx_dsp/arm/idct8x8_1_add_neon.c
    vpx_dsp/arm/idct8x8_add_neon.c
    vpx_dsp/arm/idct16x16_1_add_neon.c
    vpx_dsp/arm/idct16x16_add_neon.c
    vpx_dsp/arm/idct32x32_1_add_neon.c
    vpx_dsp/arm/idct32x32_34_add_neon.c
    vpx_dsp/arm/idct32x32_135_add_neon.c
    vpx_dsp/arm/idct32x32_add_neon.c
    vpx_dsp/arm/sad_neon.c
    vpx_dsp/arm/sad4d_neon.c
    vpx_dsp/arm/subtract_neon.c
    vpx_dsp/arm/sum_squares_neon.c
    vpx_dsp/arm/variance_neon.c
    vpx_dsp/arm/subpel_variance_neon.c
    vpx_dsp/arm/avg_pred_neon.c
    vpx_dsp/arm/quantize_neon.c
    vpx_dsp/arm/avg_neon.c
    vpx_dsp/arm/hadamard_neon.c
    vpx_dsp/arm/fdct4x4_neon.c
    vpx_dsp/arm/fdct8x8_neon.c
    vpx_dsp/arm/fdct16x16_neon.c
    vpx_dsp/arm/fdct32x32_neon.c
    vpx_dsp/arm/fdct_partial_neon.c
    vpx_dsp/arm/vpx_convolve_copy_neon.c
    vpx_dsp/arm/vpx_convolve8_neon.c
    vpx_dsp/arm/vpx_convolve_avg_neon.c
    vpx_dsp/arm/vpx_convolve_neon.c
    vpx_dsp/arm/vpx_scaled_convolve8_neon.c
  )

  if(CONFIG_ENCODERS)
    vpx_add_simd_source(VPX_DSP_SRCS NEON
      vpx_dsp/arm/sse_neon.c
    )
  endif()

  if(CONFIG_VP9_HIGHBITDEPTH)
    vpx_add_simd_source(VPX_DSP_SRCS NEON
      vpx_dsp/arm/highbd_intrapred_neon.c
      vpx_dsp/arm/highbd_loopfilter_neon.c
      vpx_dsp/arm/highbd_vpx_convolve_copy_neon.c
      vpx_dsp/arm/highbd_vpx_convolve_avg_neon.c
      vpx_dsp/arm/highbd_vpx_convolve8_neon.c
      vpx_dsp/arm/highbd_idct4x4_add_neon.c
      vpx_dsp/arm/highbd_idct8x8_add_neon.c
      vpx_dsp/arm/highbd_idct16x16_add_neon.c
      vpx_dsp/arm/highbd_idct32x32_add_neon.c
      vpx_dsp/arm/highbd_idct32x32_34_add_neon.c
      vpx_dsp/arm/highbd_idct32x32_135_add_neon.c
      vpx_dsp/arm/highbd_idct32x32_1024_add_neon.c
      vpx_dsp/arm/highbd_quantize_neon.c
      vpx_dsp/arm/highbd_hadamard_neon.c
      vpx_dsp/arm/highbd_avg_neon.c
      vpx_dsp/arm/highbd_sad4d_neon.c
      vpx_dsp/arm/highbd_sad_neon.c
      vpx_dsp/arm/highbd_avg_pred_neon.c
      vpx_dsp/arm/highbd_sse_neon.c
      vpx_dsp/arm/highbd_variance_neon.c
      vpx_dsp/arm/highbd_subpel_variance_neon.c
    )
  endif()

  if(HAVE_NEON_DOTPROD)
    vpx_add_simd_source(VPX_DSP_SRCS NEON
      vpx_dsp/arm/sad_neon_dotprod.c
      vpx_dsp/arm/sad4d_neon_dotprod.c
      vpx_dsp/arm/variance_neon_dotprod.c
      vpx_dsp/arm/subpel_variance_neon_dotprod.c
      vpx_dsp/arm/vpx_convolve8_neon_dotprod.c
    )
    if(CONFIG_ENCODERS)
      vpx_add_simd_source(VPX_DSP_SRCS NEON
        vpx_dsp/arm/sse_neon_dotprod.c
      )
    endif()
    if(CONFIG_VP9_HIGHBITDEPTH)
      vpx_add_simd_source(VPX_DSP_SRCS NEON
        vpx_dsp/arm/highbd_variance_neon_dotprod.c
      )
    endif()
  endif()

  if(HAVE_NEON_I8MM)
    vpx_add_simd_source(VPX_DSP_SRCS NEON
      vpx_dsp/arm/vpx_convolve8_neon_i8mm.c
    )
    if(CONFIG_ENCODERS)
      vpx_add_simd_source(VPX_DSP_SRCS NEON
        vpx_dsp/arm/sse_neon_i8mm.c
      )
    endif()
  endif()
endif()

if(HAVE_SVE)
  vpx_add_simd_source(VPX_DSP_SRCS SVE
    vpx_dsp/arm/sum_squares_sve.c
  )
  if(CONFIG_VP9_HIGHBITDEPTH)
    vpx_add_simd_source(VPX_DSP_SRCS SVE
      vpx_dsp/arm/highbd_vpx_convolve8_sve.c
      vpx_dsp/arm/highbd_variance_sve.c
    )
  endif()
endif()

if(HAVE_SVE2)
  if(CONFIG_VP9_HIGHBITDEPTH)
    vpx_add_simd_source(VPX_DSP_SRCS SVE2
      vpx_dsp/arm/highbd_vpx_convolve8_sve2.c
    )
  endif()
endif()

set(VPX_DSP_INCLUDES
  vpx_dsp/vpx_dsp_common.h
  vpx_dsp/prob.h
  vpx_dsp/postproc.h
  vpx_dsp/txfm_common.h
  vpx_dsp/intrapred.h
  vpx_dsp/inv_txfm.h
  vpx_dsp/vpx_filter.h
  vpx_dsp/loopfilter.h
  vpx_dsp/skin_detection.h
)

if(CONFIG_ENCODERS)
  list(APPEND VPX_DSP_INCLUDES
    vpx_dsp/bitwriter.h
    vpx_dsp/bitwriter_buffer.h
    vpx_dsp/psnr.h
    vpx_dsp/fwd_txfm.h
    vpx_dsp/quantize.h
    vpx_dsp/avg.h
    vpx_dsp/sad.h
    vpx_dsp/subtract.h
    vpx_dsp/variance.h
  )
endif()

if(CONFIG_DECODERS)
  list(APPEND VPX_DSP_INCLUDES
    vpx_dsp/bitreader.h
    vpx_dsp/bitreader_buffer.h
  )
endif()

# ============================================================================
# VPX utility
# ============================================================================
set(VPX_UTIL_SRCS
  vpx_util/vpx_thread.c
  vpx_util/vpx_write_yuv_frame.c
)

set(VPX_UTIL_INCLUDES
  vpx_util/vpx_atomics.h
  vpx_util/vpx_pthread.h
  vpx_util/vpx_thread.h
  vpx_util/endian_inl.h
  vpx_util/vpx_write_yuv_frame.h
  vpx_util/vpx_timestamp.h
)

# ============================================================================
# VP8 sources
# ============================================================================
set(VP8_COMMON_SRCS
  vp8/common/onyx.h
  vp8/common/onyxd.h
  vp8/common/alloccommon.c
  vp8/common/blockd.c
  vp8/common/dequantize.c
  vp8/common/entropy.c
  vp8/common/entropymode.c
  vp8/common/entropymv.c
  vp8/common/extend.c
  vp8/common/filter.c
  vp8/common/findnearmv.c
  vp8/common/generic/systemdependent.c
  vp8/common/idct_blk.c
  vp8/common/idctllm.c
  vp8/common/rtcd.c
  vp8/common/loopfilter_filters.c
  vp8/common/mbpitch.c
  vp8/common/modecont.c
  vp8/common/quant_common.c
  vp8/common/reconinter.c
  vp8/common/reconintra.c
  vp8/common/reconintra4x4.c
  vp8/common/setupintrarecon.c
  vp8/common/swapyv12buffer.c
  vp8/common/vp8_loopfilter.c
  vp8/common/treecoder.c
)

set(VP8_COMMON_INCLUDES
  vp8/common/ppflags.h
  vp8/common/alloccommon.h
  vp8/common/blockd.h
  vp8/common/common.h
  vp8/common/entropy.h
  vp8/common/entropymode.h
  vp8/common/entropymv.h
  vp8/common/extend.h
  vp8/common/filter.h
  vp8/common/findnearmv.h
  vp8/common/header.h
  vp8/common/invtrans.h
  vp8/common/loopfilter.h
  vp8/common/modecont.h
  vp8/common/mv.h
  vp8/common/onyxc_int.h
  vp8/common/quant_common.h
  vp8/common/reconinter.h
  vp8/common/reconintra.h
  vp8/common/reconintra4x4.h
  vp8/common/setupintrarecon.h
  vp8/common/swapyv12buffer.h
  vp8/common/systemdependent.h
  vp8/common/threading.h
  vp8/common/treecoder.h
  vp8/common/vp8_entropymodedata.h
  vp8/common/default_coef_probs.h
  vp8/common/coefupdateprobs.h
)

# VP8 postproc
if(CONFIG_POSTPROC)
  list(APPEND VP8_COMMON_SRCS
    vp8/common/mfqe.c
    vp8/common/postproc.c
  )
  list(APPEND VP8_COMMON_INCLUDES
    vp8/common/postproc.h
  )
endif()

# VP8 SIMD optimizations
if(VPX_ARCH_X86 OR VPX_ARCH_X86_64)
  if(HAVE_SSE2)
    vpx_add_simd_source(VP8_COMMON_SRCS SSE2
      vp8/common/x86/bilinear_filter_sse2.c
    )
  endif()

  if(HAVE_X86_ASM)
    list(APPEND VP8_COMMON_SRCS
      vp8/common/x86/vp8_asm_stubs.c
      vp8/common/x86/loopfilter_x86.c
    )
    if(HAVE_MMX OR HAVE_SSE2 OR HAVE_SSSE3)
      vpx_add_simd_source(VP8_COMMON_SRCS MMX
        vp8/common/x86/subpixel_mmx.asm
      )
    endif()
    if(HAVE_MMX)
      vpx_add_simd_source(VP8_COMMON_SRCS MMX
        vp8/common/x86/dequantize_mmx.asm
        vp8/common/x86/idct_blk_mmx.c
        vp8/common/x86/idctllm_mmx.asm
        vp8/common/x86/recon_mmx.asm
      )
    endif()
    if(HAVE_SSE2)
      vpx_add_simd_source(VP8_COMMON_SRCS SSE2
        vp8/common/x86/idct_blk_sse2.c
        vp8/common/x86/idctllm_sse2.asm
        vp8/common/x86/recon_sse2.asm
        vp8/common/x86/subpixel_sse2.asm
        vp8/common/x86/loopfilter_sse2.asm
        vp8/common/x86/iwalsh_sse2.asm
      )
      if(CONFIG_POSTPROC)
        vpx_add_simd_source(VP8_COMMON_SRCS SSE2
          vp8/common/x86/mfqe_sse2.asm
        )
      endif()
      if(VPX_ARCH_X86_64)
        vpx_add_simd_source(VP8_COMMON_SRCS SSE2
          vp8/common/x86/loopfilter_block_sse2_x86_64.asm
        )
      endif()
    endif()
    if(HAVE_SSSE3)
      vpx_add_simd_source(VP8_COMMON_SRCS SSSE3
        vp8/common/x86/subpixel_ssse3.asm
      )
    endif()
  endif()
endif()

if(HAVE_NEON)
  vpx_add_simd_source(VP8_COMMON_SRCS NEON
    vp8/common/arm/loopfilter_arm.c
    vp8/common/arm/neon/bilinearpredict_neon.c
    vp8/common/arm/neon/copymem_neon.c
    vp8/common/arm/neon/dc_only_idct_add_neon.c
    vp8/common/arm/neon/dequant_idct_neon.c
    vp8/common/arm/neon/dequantizeb_neon.c
    vp8/common/arm/neon/idct_blk_neon.c
    vp8/common/arm/neon/iwalsh_neon.c
    vp8/common/arm/neon/vp8_loopfilter_neon.c
    vp8/common/arm/neon/loopfiltersimplehorizontaledge_neon.c
    vp8/common/arm/neon/loopfiltersimpleverticaledge_neon.c
    vp8/common/arm/neon/mbloopfilter_neon.c
    vp8/common/arm/neon/shortidct4x4llm_neon.c
    vp8/common/arm/neon/sixtappredict_neon.c
  )
endif()

# VP8 encoder
if(CONFIG_VP8 AND CONFIG_ENCODERS)
  set(VP8_CX_SRCS
    ${VP8_COMMON_SRCS}
    vp8/vp8_cx_iface.c
    vp8/encoder/defaultcoefcounts.h
    vp8/encoder/bitstream.c
    vp8/encoder/boolhuff.c
    vp8/encoder/copy_c.c
    vp8/encoder/dct.c
    vp8/encoder/encodeframe.c
    vp8/encoder/encodeintra.c
    vp8/encoder/encodemb.c
    vp8/encoder/encodemv.c
    vp8/encoder/firstpass.c
    vp8/encoder/lookahead.c
    vp8/encoder/mcomp.c
    vp8/encoder/modecosts.c
    vp8/encoder/onyx_if.c
    vp8/encoder/pickinter.c
    vp8/encoder/picklpf.c
    vp8/encoder/vp8_quantize.c
    vp8/encoder/ratectrl.c
    vp8/encoder/rdopt.c
    vp8/encoder/segmentation.c
    vp8/encoder/tokenize.c
    vp8/encoder/treewriter.c
    vp8/encoder/temporal_filter.c
    vp8/common/vp8_skin_detection.c
  )

  set(VP8_CX_INCLUDES
    vp8/encoder/block.h
    vp8/encoder/boolhuff.h
    vp8/encoder/bitstream.h
    vp8/encoder/encodeframe.h
    vp8/encoder/encodeintra.h
    vp8/encoder/encodemb.h
    vp8/encoder/encodemv.h
    vp8/encoder/firstpass.h
    vp8/encoder/lookahead.h
    vp8/encoder/mcomp.h
    vp8/encoder/modecosts.h
    vp8/encoder/onyx_int.h
    vp8/encoder/pickinter.h
    vp8/encoder/picklpf.h
    vp8/encoder/quantize.h
    vp8/encoder/ratectrl.h
    vp8/encoder/rdopt.h
    vp8/encoder/tokenize.h
    vp8/encoder/treewriter.h
    vp8/encoder/temporal_filter.h
    vp8/encoder/segmentation.h
    vp8/common/vp8_skin_detection.h
  )

  if(CONFIG_MULTITHREAD)
    list(APPEND VP8_CX_SRCS
      vp8/encoder/ethreading.c
    )
    list(APPEND VP8_CX_INCLUDES
      vp8/encoder/ethreading.h
    )
  endif()

  if(CONFIG_TEMPORAL_DENOISING)
    list(APPEND VP8_CX_SRCS
      vp8/encoder/denoising.c
    )
    list(APPEND VP8_CX_INCLUDES
      vp8/encoder/denoising.h
    )
  endif()

  # VP8 X86 encoder optimizations
  if(VPX_ARCH_X86 OR VPX_ARCH_X86_64)
    if(HAVE_SSE2)
      vpx_add_simd_source(VP8_CX_SRCS SSE2
        vp8/encoder/x86/vp8_quantize_sse2.c
        vp8/encoder/x86/block_error_sse2.asm
        vp8/encoder/x86/copy_sse2.asm
        vp8/encoder/x86/copy_sse3.asm
        vp8/encoder/x86/dct_sse2.asm
        vp8/encoder/x86/fwalsh_sse2.asm
        vp8/encoder/x86/temporal_filter_apply_sse2.asm
        vp8/encoder/x86/vp8_enc_stubs_sse2.c
      )
    endif()

    if(HAVE_SSSE3)
      vpx_add_simd_source(VP8_CX_SRCS SSSE3
        vp8/encoder/x86/vp8_quantize_ssse3.c
      )
    endif()

    if(HAVE_SSE4_1)
      vpx_add_simd_source(VP8_CX_SRCS SSE4_1
        vp8/encoder/x86/quantize_sse4.c
      )
    endif()

    if(CONFIG_TEMPORAL_DENOISING AND HAVE_SSE2)
      vpx_add_simd_source(VP8_CX_SRCS SSE2
        vp8/encoder/x86/denoising_sse2.c
      )
    endif()
  endif()

  # VP8 ARM NEON encoder optimizations
  if(HAVE_NEON)
    vpx_add_simd_source(VP8_CX_SRCS NEON
      vp8/encoder/arm/neon/denoising_neon.c
      vp8/encoder/arm/neon/fastquantizeb_neon.c
      vp8/encoder/arm/neon/shortfdct_neon.c
      vp8/encoder/arm/neon/vp8_shortwalsh4x4_neon.c
    )
  endif()

  set(VP8_SRCS ${VP8_CX_SRCS})
  set(VP8_INCLUDES ${VP8_CX_INCLUDES} ${VP8_COMMON_INCLUDES})
endif()

# VP8 decoder
if(CONFIG_VP8 AND CONFIG_DECODERS)
  set(VP8_DX_SRCS
    ${VP8_COMMON_SRCS}
    vp8/vp8_dx_iface.c
    vp8/decoder/dboolhuff.c
    vp8/decoder/decodemv.c
    vp8/decoder/decodeframe.c
    vp8/decoder/detokenize.c
    vp8/decoder/onyxd_if.c
  )

  set(VP8_DX_INCLUDES
    vp8/decoder/dboolhuff.h
    vp8/decoder/decodemv.h
    vp8/decoder/decoderthreading.h
    vp8/decoder/detokenize.h
    vp8/decoder/onyxd_int.h
    vp8/decoder/treereader.h
  )

  if(CONFIG_MULTITHREAD)
    list(APPEND VP8_DX_SRCS
      vp8/decoder/threading.c
    )
  endif()

  if(CONFIG_ERROR_CONCEALMENT)
    list(APPEND VP8_DX_SRCS
      vp8/decoder/error_concealment.c
    )
    list(APPEND VP8_DX_INCLUDES
      vp8/decoder/ec_types.h
      vp8/decoder/error_concealment.h
    )
  endif()

  if(CONFIG_VP8 AND CONFIG_ENCODERS)
    list(APPEND VP8_SRCS ${VP8_DX_SRCS})
    list(APPEND VP8_INCLUDES ${VP8_DX_INCLUDES})
  else()
    set(VP8_SRCS ${VP8_DX_SRCS})
    set(VP8_INCLUDES ${VP8_DX_INCLUDES} ${VP8_COMMON_INCLUDES})
  endif()
endif()

# ============================================================================
# VP9 sources
# ============================================================================
set(VP9_COMMON_SRCS
  vp9/vp9_iface_common.c
  vp9/vp9_iface_common.h
  vp9/common/vp9_alloccommon.c
  vp9/common/vp9_blockd.c
  vp9/common/vp9_entropy.c
  vp9/common/vp9_entropymode.c
  vp9/common/vp9_entropymv.c
  vp9/common/vp9_frame_buffers.c
  vp9/common/vp9_idct.c
  vp9/common/vp9_filter.c
  vp9/common/vp9_loopfilter.c
  vp9/common/vp9_thread_common.c
  vp9/common/vp9_mvref_common.c
  vp9/common/vp9_pred_common.c
  vp9/common/vp9_quant_common.c
  vp9/common/vp9_reconinter.c
  vp9/common/vp9_reconintra.c
  vp9/common/vp9_common_data.c
  vp9/common/vp9_scan.c
  vp9/common/vp9_rtcd.c
  vp9/common/vp9_scale.c
  vp9/common/vp9_seg_common.c
  vp9/common/vp9_tile_common.c
)

set(VP9_COMMON_INCLUDES
  vp9/vp9_iface_common.h
  vp9/common/vp9_ppflags.h
  vp9/common/vp9_alloccommon.h
  vp9/common/vp9_blockd.h
  vp9/common/vp9_common.h
  vp9/common/vp9_entropy.h
  vp9/common/vp9_entropymode.h
  vp9/common/vp9_entropymv.h
  vp9/common/vp9_enums.h
  vp9/common/vp9_filter.h
  vp9/common/vp9_frame_buffers.h
  vp9/common/vp9_idct.h
  vp9/common/vp9_loopfilter.h
  vp9/common/vp9_thread_common.h
  vp9/common/vp9_mv.h
  vp9/common/vp9_onyxc_int.h
  vp9/common/vp9_pred_common.h
  vp9/common/vp9_quant_common.h
  vp9/common/vp9_reconinter.h
  vp9/common/vp9_reconintra.h
  vp9/common/vp9_rtcd_defs.pl
  vp9/common/vp9_scale.h
  vp9/common/vp9_seg_common.h
  vp9/common/vp9_tile_common.h
  vp9/common/vp9_common_data.h
  vp9/common/vp9_scan.h
)

# VP9 postproc
if(CONFIG_VP9_POSTPROC)
  list(APPEND VP9_COMMON_SRCS
    vp9/common/vp9_postproc.c
    vp9/common/vp9_mfqe.c
  )
  list(APPEND VP9_COMMON_INCLUDES
    vp9/common/vp9_postproc.h
    vp9/common/vp9_mfqe.h
  )
endif()

# VP9 SIMD optimizations
if(VPX_ARCH_X86 OR VPX_ARCH_X86_64)
  if(HAVE_SSE2)
    vpx_add_simd_source(VP9_COMMON_SRCS SSE2
      vp9/common/x86/vp9_idct_intrin_sse2.c
    )
    if(CONFIG_VP9_POSTPROC AND HAVE_X86_ASM)
      vpx_add_simd_source(VP9_COMMON_SRCS SSE2
        vp9/common/x86/vp9_mfqe_sse2.asm
      )
    endif()
  endif()
  if(HAVE_SSE4_1 AND CONFIG_VP9_HIGHBITDEPTH)
    vpx_add_simd_source(VP9_COMMON_SRCS SSE4_1
      vp9/common/x86/vp9_highbd_iht4x4_add_sse4.c
      vp9/common/x86/vp9_highbd_iht8x8_add_sse4.c
      vp9/common/x86/vp9_highbd_iht16x16_add_sse4.c
    )
  endif()
endif()

if(HAVE_NEON)
  vpx_add_simd_source(VP9_COMMON_SRCS NEON
    vp9/common/arm/neon/vp9_iht4x4_add_neon.c
    vp9/common/arm/neon/vp9_iht8x8_add_neon.c
    vp9/common/arm/neon/vp9_iht16x16_add_neon.c
  )
  if(CONFIG_VP9_HIGHBITDEPTH)
    vpx_add_simd_source(VP9_COMMON_SRCS NEON
      vp9/common/arm/neon/vp9_highbd_iht4x4_add_neon.c
      vp9/common/arm/neon/vp9_highbd_iht8x8_add_neon.c
      vp9/common/arm/neon/vp9_highbd_iht16x16_add_neon.c
    )
  endif()
endif()

# VP9 encoder
if(CONFIG_VP9 AND CONFIG_ENCODERS)
  set(VP9_CX_SRCS
    ${VP9_COMMON_SRCS}
    vp9/vp9_cx_iface.c
    vp9/encoder/vp9_bitstream.c
    vp9/encoder/vp9_context_tree.c
    vp9/encoder/vp9_cost.c
    vp9/encoder/vp9_dct.c
    vp9/encoder/vp9_encodeframe.c
    vp9/encoder/vp9_encodemb.c
    vp9/encoder/vp9_encodemv.c
    vp9/encoder/vp9_ethread.c
    vp9/encoder/vp9_extend.c
    vp9/encoder/vp9_firstpass.c
    vp9/encoder/vp9_frame_scale.c
    vp9/encoder/vp9_lookahead.c
    vp9/encoder/vp9_mcomp.c
    vp9/encoder/vp9_multi_thread.c
    vp9/encoder/vp9_encoder.c
    vp9/encoder/vp9_picklpf.c
    vp9/encoder/vp9_quantize.c
    vp9/encoder/vp9_ratectrl.c
    vp9/encoder/vp9_rd.c
    vp9/encoder/vp9_rdopt.c
    vp9/encoder/vp9_pickmode.c
    vp9/encoder/vp9_segmentation.c
    vp9/encoder/vp9_speed_features.c
    vp9/encoder/vp9_subexp.c
    vp9/encoder/vp9_svc_layercontext.c
    vp9/encoder/vp9_resize.c
    vp9/encoder/vp9_tokenize.c
    vp9/encoder/vp9_treewriter.c
    vp9/encoder/vp9_aq_variance.c
    vp9/encoder/vp9_aq_360.c
    vp9/encoder/vp9_aq_cyclicrefresh.c
    vp9/encoder/vp9_aq_complexity.c
    vp9/encoder/vp9_alt_ref_aq.c
    vp9/encoder/vp9_skin_detection.c
    vp9/encoder/vp9_noise_estimate.c
    vp9/encoder/vp9_ext_ratectrl.c
    vp9/encoder/vp9_temporal_filter.c
    vp9/encoder/vp9_tpl_model.c
    vp9/encoder/vp9_mbgraph.c
  )

  if(CONFIG_VP9_TEMPORAL_DENOISING)
    list(APPEND VP9_CX_SRCS
      vp9/encoder/vp9_denoiser.c
    )
  endif()

  set(VP9_CX_INCLUDES
    vp9/vp9_cx_iface.h
    vp9/encoder/vp9_context_tree.h
    vp9/encoder/vp9_cost.h
    vp9/encoder/vp9_denoiser.h
    vp9/encoder/vp9_encodeframe.h
    vp9/encoder/vp9_encodemb.h
    vp9/encoder/vp9_encodemv.h
    vp9/encoder/vp9_ethread.h
    vp9/encoder/vp9_block.h
    vp9/encoder/vp9_bitstream.h
    vp9/encoder/vp9_encodemb.h
    vp9/encoder/vp9_encodemv.h
    vp9/encoder/vp9_extend.h
    vp9/encoder/vp9_firstpass.h
    vp9/encoder/vp9_firstpass_stats.h
    vp9/encoder/vp9_job_queue.h
    vp9/encoder/vp9_lookahead.h
    vp9/encoder/vp9_mcomp.h
    vp9/encoder/vp9_multi_thread.h
    vp9/encoder/vp9_encoder.h
    vp9/encoder/vp9_quantize.h
    vp9/encoder/vp9_ratectrl.h
    vp9/encoder/vp9_rd.h
    vp9/encoder/vp9_rdopt.h
    vp9/encoder/vp9_pickmode.h
    vp9/encoder/vp9_svc_layercontext.h
    vp9/encoder/vp9_tokenize.h
    vp9/encoder/vp9_treewriter.h
    vp9/encoder/vp9_partition_models.h
    vp9/encoder/vp9_segmentation.h
    vp9/encoder/vp9_speed_features.h
    vp9/encoder/vp9_subexp.h
    vp9/encoder/vp9_resize.h
    vp9/encoder/vp9_blockiness.h
    vp9/encoder/vp9_alt_ref_aq.h
    vp9/encoder/vp9_skin_detection.h
    vp9/encoder/vp9_noise_estimate.h
    vp9/encoder/vp9_ext_ratectrl.h
    vp9/encoder/vp9_temporal_filter.h
    vp9/encoder/vp9_tpl_model.h
    vp9/encoder/vp9_mbgraph.h
  )

  # VP9 X86 encoder optimizations
  if(VPX_ARCH_X86 OR VPX_ARCH_X86_64)
    if(HAVE_SSE2)
      vpx_add_simd_source(VP9_CX_SRCS SSE2
        vp9/encoder/x86/vp9_quantize_sse2.c
        vp9/encoder/x86/vp9_dct_sse2.asm
        vp9/encoder/x86/vp9_error_sse2.asm
        vp9/encoder/x86/vp9_dct_intrin_sse2.c
        vp9/encoder/x86/vp9_denoiser_sse2.c
      )
      if(CONFIG_VP9_HIGHBITDEPTH)
        vpx_add_simd_source(VP9_CX_SRCS SSE2
          vp9/encoder/x86/vp9_highbd_block_error_intrin_sse2.c
        )
      endif()
    endif()

    if(HAVE_SSSE3)
      vpx_add_simd_source(VP9_CX_SRCS SSSE3
        vp9/encoder/x86/vp9_quantize_ssse3.c
        vp9/encoder/x86/vp9_frame_scale_ssse3.c
        vp9/encoder/x86/temporal_filter_ssse3.c
      )
      if(CONFIG_VP9_HIGHBITDEPTH)
        vpx_add_simd_source(VP9_CX_SRCS SSSE3
          vp9/encoder/x86/highbd_temporal_filter_ssse3.c
        )
      endif()
    endif()

    if(HAVE_SSE4_1)
      vpx_add_simd_source(VP9_CX_SRCS SSE4_1
        vp9/encoder/x86/temporal_filter_sse4.c
      )
      if(CONFIG_VP9_HIGHBITDEPTH)
        vpx_add_simd_source(VP9_CX_SRCS SSE4_1
          vp9/encoder/x86/highbd_temporal_filter_sse4.c
        )
      endif()
    endif()

    if(HAVE_AVX2)
      vpx_add_simd_source(VP9_CX_SRCS AVX2
        vp9/encoder/x86/vp9_quantize_avx2.c
        vp9/encoder/x86/vp9_error_avx2.c
        vp9/encoder/x86/temporal_filter_avx2.c
      )
      if(CONFIG_VP9_HIGHBITDEPTH)
        vpx_add_simd_source(VP9_CX_SRCS AVX2
          vp9/encoder/x86/highbd_temporal_filter_avx2.c
        )
      endif()
    endif()
  endif()

  # VP9 ARM NEON encoder optimizations
  if(HAVE_NEON)
    vpx_add_simd_source(VP9_CX_SRCS NEON
      vp9/encoder/arm/neon/vp9_dct_neon.c
      vp9/encoder/arm/neon/vp9_quantize_neon.c
      vp9/encoder/arm/neon/vp9_error_neon.c
      vp9/encoder/arm/neon/vp9_frame_scale_neon.c
      vp9/encoder/arm/neon/vp9_diamond_search_sad_neon.c
      vp9/encoder/arm/neon/vp9_temporal_filter_neon.c
      vp9/encoder/arm/neon/vp9_denoiser_neon.c
    )

    if(CONFIG_VP9_HIGHBITDEPTH)
      vpx_add_simd_source(VP9_CX_SRCS NEON
        vp9/encoder/arm/neon/vp9_highbd_error_neon.c
        vp9/encoder/arm/neon/vp9_highbd_temporal_filter_neon.c
      )
    endif()

    if(HAVE_NEON_DOTPROD)
      vpx_add_simd_source(VP9_CX_SRCS NEON
        vp9/encoder/arm/neon/vp9_temporal_filter_neon_dotprod.c
      )
    endif()

    if(HAVE_NEON_I8MM)
      vpx_add_simd_source(VP9_CX_SRCS NEON
        vp9/encoder/arm/neon/vp9_temporal_filter_neon_i8mm.c
      )
    endif()

    if(HAVE_SVE)
      vpx_add_simd_source(VP9_CX_SRCS SVE
        vp9/encoder/arm/neon/vp9_error_sve.c
      )
    endif()

    if(HAVE_SVE2 AND CONFIG_VP9_HIGHBITDEPTH)
      vpx_add_simd_source(VP9_CX_SRCS SVE2
        vp9/encoder/arm/neon/vp9_highbd_temporal_filter_sve2.c
      )
    endif()
  endif()

  set(VP9_SRCS ${VP9_CX_SRCS})
  set(VP9_INCLUDES ${VP9_CX_INCLUDES} ${VP9_COMMON_INCLUDES})
endif()

# VP9 decoder
if(CONFIG_VP9 AND CONFIG_DECODERS)
  set(VP9_DX_SRCS
    ${VP9_COMMON_SRCS}
    vp9/vp9_dx_iface.c
    vp9/decoder/vp9_decodemv.c
    vp9/decoder/vp9_decodeframe.c
    vp9/decoder/vp9_detokenize.c
    vp9/decoder/vp9_decoder.c
    vp9/decoder/vp9_dsubexp.c
    vp9/decoder/vp9_job_queue.c
  )

  set(VP9_DX_INCLUDES
    vp9/vp9_dx_iface.h
    vp9/decoder/vp9_decodemv.h
    vp9/decoder/vp9_decodeframe.h
    vp9/decoder/vp9_detokenize.h
    vp9/decoder/vp9_decoder.h
    vp9/decoder/vp9_dsubexp.h
    vp9/decoder/vp9_job_queue.h
  )

  if(CONFIG_VP9 AND CONFIG_ENCODERS)
    list(APPEND VP9_SRCS ${VP9_DX_SRCS})
    list(APPEND VP9_INCLUDES ${VP9_DX_INCLUDES})
  else()
    set(VP9_SRCS ${VP9_DX_SRCS})
    set(VP9_INCLUDES ${VP9_DX_INCLUDES} ${VP9_COMMON_INCLUDES})
  endif()
endif()

# ============================================================================
# Collect all sources
# ============================================================================
set(LIBVPX_SRCS "")
list(APPEND LIBVPX_SRCS ${VPX_SRCS})
list(APPEND LIBVPX_SRCS ${VPX_MEM_SRCS})
list(APPEND LIBVPX_SRCS ${VPX_SCALE_SRCS})
list(APPEND LIBVPX_SRCS ${VPX_PORTS_SRCS})
list(APPEND LIBVPX_SRCS ${VPX_DSP_SRCS})
list(APPEND LIBVPX_SRCS ${VPX_UTIL_SRCS})

if(CONFIG_VP8)
  list(APPEND LIBVPX_SRCS ${VP8_SRCS})
endif()

if(CONFIG_VP9)
  list(APPEND LIBVPX_SRCS ${VP9_SRCS})
endif()

# Collect all includes for installation
set(LIBVPX_INCLUDES "")
list(APPEND LIBVPX_INCLUDES ${VPX_INCLUDES})
list(APPEND LIBVPX_INCLUDES ${VPX_MEM_INCLUDES})
list(APPEND LIBVPX_INCLUDES ${VPX_SCALE_INCLUDES})
list(APPEND LIBVPX_INCLUDES ${VPX_PORTS_INCLUDES})
list(APPEND LIBVPX_INCLUDES ${VPX_DSP_INCLUDES})
list(APPEND LIBVPX_INCLUDES ${VPX_UTIL_INCLUDES})


if(CONFIG_VP8)
  list(APPEND LIBVPX_INCLUDES ${VP8_INCLUDES})
endif()

if(CONFIG_VP9)
  list(APPEND LIBVPX_INCLUDES ${VP9_INCLUDES})
endif()

set(LIBVPX_EXISTING_INCLUDES "")
foreach(hdr ${LIBVPX_INCLUDES})
  if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${hdr}")
    list(APPEND LIBVPX_EXISTING_INCLUDES ${hdr})
  endif()
endforeach()
set(LIBVPX_INCLUDES ${LIBVPX_EXISTING_INCLUDES})

# Add generated RTCD headers to sources for dependency tracking
list(APPEND LIBVPX_SRCS ${RTCD_TARGETS})

# ============================================================================
# Library definition
# ============================================================================

# Include directories
set(VPX_PUBLIC_INCLUDES
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${CMAKE_CURRENT_BINARY_DIR}
)

# Unified library target
if(CONFIG_SHARED)
  add_library(vpx SHARED ${LIBVPX_SRCS})
  target_compile_definitions(vpx PRIVATE VPX_EXPORTS)
  if(WIN32)
    set_target_properties(vpx PROPERTIES
      WINDOWS_EXPORT_ALL_SYMBOLS ON
    )
  endif()
else()
  add_library(vpx STATIC ${LIBVPX_SRCS})
endif()

if(MSVC)
  # Forcefully clear the MSVC runtime library property for the target to avoid NASM conflict.
  set_target_properties(vpx PROPERTIES MSVC_RUNTIME_LIBRARY "")
endif()

add_library(vpx::vpx ALIAS vpx)

target_include_directories(vpx PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/vpx_dsp>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/vpx_scale>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/vp8/common>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/vp9/common>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

if(MSVC AND (VPX_ARCH_ARM OR VPX_ARCH_AARCH64))
  target_include_directories(vpx BEFORE PRIVATE "${MSVC_ARM_NEON_COMPAT_DIR}")
endif()

target_compile_definitions(vpx PRIVATE
  $<$<BOOL:${CONFIG_VP8}>:CONFIG_VP8=1>
  $<$<BOOL:${CONFIG_VP9}>:CONFIG_VP9=1>
  $<$<BOOL:${CONFIG_ENCODERS}>:CONFIG_ENCODERS=1>
  $<$<BOOL:${CONFIG_DECODERS}>:CONFIG_DECODERS=1>
  $<$<BOOL:${CONFIG_MULTITHREAD}>:CONFIG_MULTITHREAD=1>
  $<$<BOOL:${CONFIG_VP9_HIGHBITDEPTH}>:CONFIG_VP9_HIGHBITDEPTH=1>
  $<$<BOOL:${CONFIG_POSTPROC}>:CONFIG_POSTPROC=1>
  $<$<BOOL:${CONFIG_SPATIAL_RESAMPLING}>:CONFIG_SPATIAL_RESAMPLING=1>
  $<$<BOOL:${CONFIG_TEMPORAL_DENOISING}>:CONFIG_TEMPORAL_DENOISING=1>
  $<$<BOOL:${CONFIG_VP9_TEMPORAL_DENOISING}>:CONFIG_VP9_TEMPORAL_DENOISING=1>
  $<$<BOOL:${CONFIG_ERROR_CONCEALMENT}>:CONFIG_ERROR_CONCEALMENT=1>
  $<$<BOOL:${CONFIG_REALTIME_ONLY}>:CONFIG_REALTIME_ONLY=1>
  $<$<BOOL:${CONFIG_ONTHEFLY_BITPACKING}>:CONFIG_ONTHEFLY_BITPACKING=1>
  $<$<BOOL:${CONFIG_OS_SUPPORT}>:CONFIG_OS_SUPPORT=1>
  $<$<BOOL:${CONFIG_INTERNAL_STATS}>:CONFIG_INTERNAL_STATS=1>
)

if(HAVE_PTHREAD_H)
  target_compile_definitions(vpx PRIVATE HAVE_PTHREAD_H=1)
endif()

if(Threads_FOUND)
  target_link_libraries(vpx PRIVATE Threads::Threads)
endif()

if(NOT WIN32)
  target_link_libraries(vpx PRIVATE m)
endif()

set_target_properties(vpx PROPERTIES
  OUTPUT_NAME vpx
  VERSION ${PROJECT_VERSION}
  SOVERSION ${PROJECT_VERSION_MAJOR}
  ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}"
  LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}"
  RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}"
)

set(LIBVPX_TARGET vpx)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

set(VPX_CONFIG_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/vpx")

configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/vpxConfig.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/vpxConfig.cmake"
  INSTALL_DESTINATION "${VPX_CONFIG_INSTALL_DIR}"
)

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/vpxConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion
)

# Install libraries
install(TARGETS vpx
  EXPORT vpxTargets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Export targets
install(EXPORT vpxTargets
  NAMESPACE vpx::
  DESTINATION ${VPX_CONFIG_INSTALL_DIR}
)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/vpxConfig.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/vpxConfigVersion.cmake"
  DESTINATION ${VPX_CONFIG_INSTALL_DIR}
)

# Install public headers
foreach(hdr ${LIBVPX_INCLUDES})
  # Get relative path
  file(RELATIVE_PATH rel_path ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/${hdr})
  get_filename_component(hdr_dir ${rel_path} DIRECTORY)

  install(FILES ${hdr}
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${hdr_dir}
  )
endforeach()


# Install generated config header
install(FILES ${CONFIG_H}
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/vpx
)

# ============================================================================
# pkg-config file
# ============================================================================
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/vpx.pc.in"
  "${CMAKE_CURRENT_BINARY_DIR}/vpx.pc"
  @ONLY
)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/vpx.pc"
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

# ============================================================================
# Examples
# ============================================================================
if(CONFIG_EXAMPLES)
set(EXAMPLE_SRCS
  args.c
  y4minput.c
  y4menc.c
  ivfdec.c
  ivfenc.c
  tools_common.c
  video_reader.c
  video_writer.c
  warnings.c
  vpxstats.c
  rate_hist.c
)

add_library(vpx_examples_common STATIC ${EXAMPLE_SRCS})
target_link_libraries(vpx_examples_common PRIVATE vpx)

macro(add_vpx_example name source)
  add_executable(${name} ${source})
  target_link_libraries(${name} PRIVATE vpx_examples_common vpx)
  set_target_properties(${name} PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/examples"
  )
endmacro()

if(CONFIG_DECODERS)
  add_vpx_example(simple_decoder examples/simple_decoder.c)
endif()

if(CONFIG_ENCODERS)
  add_vpx_example(simple_encoder examples/simple_encoder.c)
endif()
endif()

# ============================================================================
# Summary
# ============================================================================
message(STATUS "")
message(STATUS "libvpx ${PROJECT_VERSION} Configuration:")
message(STATUS "  Architecture:      ${CMAKE_SYSTEM_PROCESSOR}")
message(STATUS "  VP8:               ${CONFIG_VP8}")
message(STATUS "  VP9:               ${CONFIG_VP9}")
message(STATUS "  Encoders:          ${CONFIG_ENCODERS}")
message(STATUS "  Decoders:          ${CONFIG_DECODERS}")
message(STATUS "  Multithread:       ${CONFIG_MULTITHREAD}")
message(STATUS "  Static library:    ${CONFIG_STATIC}")
message(STATUS "  Shared library:    ${CONFIG_SHARED}")
message(STATUS "  VP9 HighBitDepth:  ${CONFIG_VP9_HIGHBITDEPTH}")
message(STATUS "  Postproc:          ${CONFIG_POSTPROC}")
message(STATUS "")
message(STATUS "  SIMD Features:")
message(STATUS "    MMX:             ${HAVE_MMX}")
message(STATUS "    SSE2:            ${HAVE_SSE2}")
message(STATUS "    SSE3:            ${HAVE_SSE3}")
message(STATUS "    SSSE3:           ${HAVE_SSSE3}")
message(STATUS "    SSE4.1:          ${HAVE_SSE4_1}")
message(STATUS "    AVX:             ${HAVE_AVX}")
message(STATUS "    AVX2:            ${HAVE_AVX2}")
message(STATUS "    AVX512:          ${HAVE_AVX512}")
message(STATUS "    NEON:            ${HAVE_NEON}")
message(STATUS "    NEON dotprod:    ${HAVE_NEON_DOTPROD}")
message(STATUS "    NEON i8mm:       ${HAVE_NEON_I8MM}")
message(STATUS "    SVE:             ${HAVE_SVE}")
message(STATUS "    SVE2:            ${HAVE_SVE2}")
message(STATUS "")
