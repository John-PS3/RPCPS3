if (NOT DEFINED FFMPEG_CORE_NAME)
	set(FFMPEG_CORE_NAME ffmpeg)
endif()

add_library(${FFMPEG_CORE_NAME} INTERFACE)

target_include_directories(${FFMPEG_CORE_NAME} INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/include")

# Detect OS
if (WIN32 AND NOT MINGW)
	set(FFMPEG_PREBUILTS_NAME "ffmpeg-windows")
elseif (APPLE)
	set(FFMPEG_PREBUILTS_NAME "ffmpeg-macos")
elseif (UNIX)
	set(FFMPEG_PREBUILTS_NAME "ffmpeg-linux")
else ()
	message(FATAL_ERROR "Unsupported OS.")
endif ()

# Detect Architecture
if(NOT DEFINED ARCHITECTURE)
	message(FATAL_ERROR "ARCHITECTURE variable is not set up")
elseif (ARCHITECTURE STREQUAL "x86_64")
	set(FFMPEG_PREBUILTS_NAME "${FFMPEG_PREBUILTS_NAME}-x64.zip")
elseif(ARCHITECTURE STREQUAL "arm64")
	set(FFMPEG_PREBUILTS_NAME "${FFMPEG_PREBUILTS_NAME}-arm64.zip")
elseif(APPLE AND "x86_64" IN_LIST ARCHITECTURE AND "arm64" IN_LIST ARCHITECTURE) # macOS universal
	set(FFMPEG_PREBUILTS_NAME "${FFMPEG_PREBUILTS_NAME}-universal.zip")
else ()
	message(FATAL_ERROR "Unsupported architecture.")
endif()

# Compute current short git commit SHA
execute_process(
	COMMAND git rev-parse --short HEAD
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	OUTPUT_VARIABLE FFMPEG_GIT_SHA
	OUTPUT_STRIP_TRAILING_WHITESPACE)

# Download prebuilt ffmpeg
if(NOT EXISTS "${CMAKE_BINARY_DIR}/3rdparty/ffmpeg.zip")
	message(STATUS "Downloading FFMPEG prebuilts...")
	file(DOWNLOAD https://github.com/RPCS3/ffmpeg-core/releases/download/${FFMPEG_GIT_SHA}/${FFMPEG_PREBUILTS_NAME}
		"${CMAKE_BINARY_DIR}/3rdparty/ffmpeg.zip" SHOW_PROGRESS
		STATUS FILE_STATUS)
	list(GET FILE_STATUS 0 STATUS_CODE)
	if (NOT STATUS_CODE EQUAL 0)
		file(REMOVE "${CMAKE_BINARY_DIR}/3rdparty/ffmpeg.zip") # CMake create 0 byte file even if URL is invalid. So need to delete it.
		message(FATAL_ERROR "No FFMPEG prebuilt found with corresponding commit SHA (${FFMPEG_GIT_SHA})")
	endif()
endif()

if(NOT EXISTS "${CMAKE_BINARY_DIR}/3rdparty/ffmpeg/lib")
	file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/3rdparty/ffmpeg/lib")
	execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf "${CMAKE_BINARY_DIR}/3rdparty/ffmpeg.zip"
		WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/3rdparty/ffmpeg/lib")
endif()

set(LIB_PREFIX "lib")
set(LIB_EXT "a")
if (WIN32 AND NOT MINGW)
	set(LIB_PREFIX "")
	set(LIB_EXT "lib")
	target_link_libraries(${FFMPEG_CORE_NAME} INTERFACE
		"psapi;strmiids;uuid;oleaut32;shlwapi;ws2_32;ole32;user32;bcrypt")
elseif (APPLE)
	target_link_libraries(${FFMPEG_CORE_NAME} INTERFACE
		"-framework CoreServices" "-framework CoreFoundation" "-framework AudioUnit"
		"-framework AudioToolbox" "-framework CoreAudio" "-framework CoreMedia"
		"-framework VideoToolbox" "-framework CoreVideo" "-framework Security")
endif ()

target_link_libraries(${FFMPEG_CORE_NAME} INTERFACE
	"${CMAKE_BINARY_DIR}/3rdparty/ffmpeg/lib/${LIB_PREFIX}avformat.${LIB_EXT}"
	"${CMAKE_BINARY_DIR}/3rdparty/ffmpeg/lib/${LIB_PREFIX}avcodec.${LIB_EXT}"
	"${CMAKE_BINARY_DIR}/3rdparty/ffmpeg/lib/${LIB_PREFIX}swscale.${LIB_EXT}"
	"${CMAKE_BINARY_DIR}/3rdparty/ffmpeg/lib/${LIB_PREFIX}avutil.${LIB_EXT}"
	"${CMAKE_BINARY_DIR}/3rdparty/ffmpeg/lib/${LIB_PREFIX}avfilter.${LIB_EXT}"
	"${CMAKE_BINARY_DIR}/3rdparty/ffmpeg/lib/${LIB_PREFIX}swresample.${LIB_EXT}")
