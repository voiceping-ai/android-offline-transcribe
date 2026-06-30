cmake_minimum_required(VERSION 3.22.1)
project(native_engines)

set(CMAKE_CXX_STANDARD 17)

# ---- Qwen ASR core library (mel spectrogram + tokenizer, used by ONNX engine) ----
set(QWEN_ASR_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../../../qwen-asr)

if(ANDROID_ABI STREQUAL "armeabi-v7a")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpu=neon -mfloat-abi=softfp")
endif()

add_library(qwen_asr STATIC
    ${QWEN_ASR_DIR}/qwen_asr.c
    ${QWEN_ASR_DIR}/qwen_asr_kernels.c
    ${QWEN_ASR_DIR}/qwen_asr_kernels_generic.c
    ${QWEN_ASR_DIR}/qwen_asr_kernels_neon.c
    ${QWEN_ASR_DIR}/qwen_asr_kernels_avx.c
    ${QWEN_ASR_DIR}/qwen_asr_audio.c
    ${QWEN_ASR_DIR}/qwen_asr_encoder.c
    ${QWEN_ASR_DIR}/qwen_asr_decoder.c
    ${QWEN_ASR_DIR}/qwen_asr_tokenizer.c
    ${QWEN_ASR_DIR}/qwen_asr_safetensors.c
)

target_include_directories(qwen_asr PUBLIC ${QWEN_ASR_DIR})

target_compile_options(qwen_asr PRIVATE
    -O3 -ffast-math
)

# ---- Qwen ASR JNI (Pure C/NEON inference) ----
add_library(qwen_asr_jni SHARED qwen_asr_jni.cpp)

target_include_directories(qwen_asr_jni PRIVATE ${QWEN_ASR_DIR})

target_link_libraries(qwen_asr_jni
    qwen_asr
    log
)

# ---- Qwen ONNX library (ONNX Runtime inference) ----
add_library(qwen_asr_onnx STATIC
    ${QWEN_ASR_DIR}/qwen_asr_onnx.c
)

target_include_directories(qwen_asr_onnx PRIVATE
    ${QWEN_ASR_DIR}
    ${QWEN_ASR_DIR}/ort
)

target_compile_options(qwen_asr_onnx PRIVATE
    -O3 -ffast-math
)

# Build Qwen ONNX JNI bridge
# Uses dlopen to load libonnxruntime.so at runtime (from sherpa-onnx AAR)
add_library(qwen_onnx_jni SHARED qwen_onnx_jni.cpp)

target_include_directories(qwen_onnx_jni PRIVATE ${QWEN_ASR_DIR})

target_link_libraries(qwen_onnx_jni
    qwen_asr_onnx
    qwen_asr          # for mel spectrogram + tokenizer
    dl                 # for dlopen/dlsym (loads libonnxruntime.so at runtime)
    log
)

# ---- whisper.cpp (direct GGML inference, matching iOS CactusKit approach) ----
set(WHISPER_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../../../whisper.cpp)

include(FetchContent)
FetchContent_Declare(ggml SOURCE_DIR ${WHISPER_DIR}/ggml)
FetchContent_MakeAvailable(ggml)

add_library(whisper_lib STATIC
    ${WHISPER_DIR}/src/whisper.cpp
)

target_include_directories(whisper_lib PUBLIC
    ${WHISPER_DIR}/include
    ${WHISPER_DIR}/src
    ${WHISPER_DIR}/ggml/include
    ${WHISPER_DIR}/ggml/src
    ${WHISPER_DIR}/ggml/src/ggml-cpu
)

target_compile_definitions(whisper_lib PUBLIC GGML_USE_CPU)
target_compile_definitions(whisper_lib PRIVATE WHISPER_VERSION="1.8.3")
target_compile_options(whisper_lib PRIVATE -O3)
target_link_libraries(whisper_lib PUBLIC ggml)

find_package(Threads REQUIRED)
target_link_libraries(whisper_lib PUBLIC Threads::Threads)

# ---- Whisper JNI bridge ----
add_library(whisper_jni SHARED whisper_jni.c)

target_include_directories(whisper_jni PRIVATE
    ${WHISPER_DIR}/include
)

target_link_libraries(whisper_jni
    whisper_lib
    android
    log
)
