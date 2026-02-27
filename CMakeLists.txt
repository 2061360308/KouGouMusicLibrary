file(MAKE_DIRECTORY ${CMAKE_SOURCE_DIR}/bundle)
file(MAKE_DIRECTORY ${CMAKE_SOURCE_DIR}/bundle/js)
file(MAKE_DIRECTORY ${CMAKE_SOURCE_DIR}/bundle/c)

cmake_minimum_required(VERSION 3.10)
project(kugouC)

add_subdirectory(quickjs)

# file(GLOB JS_SRC_FILES "${CMAKE_SOURCE_DIR}/src/js/*.js")
# add_custom_command(
#     OUTPUT ${CMAKE_SOURCE_DIR}/js_bundle/app.bundle.js
#     COMMAND npx webpack --config ${CMAKE_SOURCE_DIR}/webpack.config.js
#     WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
#     DEPENDS ${JS_SRC_FILES}
#     COMMENT "Auto bundle JS when src/js changes"
# )

# 使用 qjsc 把 js_bundle/app.bundle.js 编译成 c_bundle/app.bundle.c
add_custom_command(
	OUTPUT ${CMAKE_SOURCE_DIR}/bundle/c/app.bundle.c
	COMMAND $<TARGET_FILE:qjsc> -o ${CMAKE_SOURCE_DIR}/bundle/c/app.bundle.c ${CMAKE_SOURCE_DIR}/bundle/js/app.bundle.js
	DEPENDS qjsc ${CMAKE_SOURCE_DIR}/bundle/js/app.bundle.js
	COMMENT "Using qjsc to compile app.bundle.js"
)

add_custom_target(gen_app_bundle_c ALL
	DEPENDS ${CMAKE_SOURCE_DIR}/bundle/c/app.bundle.c
)

add_library(kugou_music_api STATIC
	${CMAKE_SOURCE_DIR}/src/c/main.c
	${CMAKE_SOURCE_DIR}/src/c/http.c
	${CMAKE_SOURCE_DIR}/src/c/tinycthread.c
	${CMAKE_SOURCE_DIR}/src/c/kugou_music_api.c
	${CMAKE_SOURCE_DIR}/bundle/c/app.bundle.c
)

target_include_directories(kugou_music_api
	PRIVATE
		${CMAKE_SOURCE_DIR}/quickjs
		${CMAKE_SOURCE_DIR}/src/include
)
# 链接 quickjs 提供的静态库，解决 JS_* 符号
target_link_libraries(kugou_music_api PRIVATE qjs qjs-libc)

# 告诉 CMake 在 vcpkg_installed 中查找包的 Config.cmake
list(APPEND CMAKE_PREFIX_PATH "${CMAKE_SOURCE_DIR}/vcpkg_installed/x64-windows")

find_package(CURL CONFIG REQUIRED)
target_link_libraries(kugou_music_api PRIVATE CURL::libcurl)

# 通过 vcpkg 安装的 libuv（Config 模式）
# find_package(libuv CONFIG REQUIRED)
# target_link_libraries(kugou_music_api
# 	PRIVATE $<IF:$<TARGET_EXISTS:libuv::uv_a>,libuv::uv_a,libuv::uv>
# )

add_executable(test ${CMAKE_SOURCE_DIR}/src/c/test.c)
target_include_directories(test
	PRIVATE
		${CMAKE_SOURCE_DIR}/src/include
)
target_link_libraries(test PRIVATE kugou_music_api)