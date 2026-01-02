cmake_minimum_required(VERSION 3.15)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_LIST_DIR}/cmake/Modules")
project(soem_rsl C)

find_package(ament_cmake REQUIRED)

# -------------------------
# OS selection (kept)
# -------------------------
if(WIN32)
  set(OS "win32")
  include_directories(oshw/win32/wpcap/Include)
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    link_directories(${CMAKE_CURRENT_LIST_DIR}/oshw/win32/wpcap/Lib/x64)
  elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
    link_directories(${CMAKE_CURRENT_LIST_DIR}/oshw/win32/wpcap/Lib)
  endif()
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /D _CRT_SECURE_NO_WARNINGS")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /WX")
  set(OS_LIBS wpcap.lib Packet.lib Ws2_32.lib Winmm.lib)
elseif(UNIX AND NOT APPLE)
  set(OS "linux")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror")
  set(OS_LIBS pthread rt)
elseif(APPLE)
  set(OS "macosx")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror")
  set(OS_LIBS pthread pcap)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "rt-kernel")
  set(OS "rtk")
  message(STATUS "ARCH is ${ARCH}")
  message(STATUS "BSP is ${BSP}")
  include_directories(oshw/${OS}/${ARCH})
  file(GLOB OSHW_EXTRA_SOURCES oshw/${OS}/${ARCH}/*.c)
  set(OSHW_SOURCES "${OS_HW_SOURCES} ${OSHW_ARCHSOURCES}")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Werror")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unused-but-set-variable")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unused-function")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-format")
  set(OS_LIBS "-Wl,--start-group -l${BSP} -l${ARCH} -lkern -ldev -lsio -lblock -lfs -lusb -llwip -leth -li2c -lrtc -lcan -lnand -lspi -lnor -lpwm -ladc -ltrace -lc -lm -Wl,--end-group")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "rtems")
  message(STATUS "Building for RTEMS")
  set(OS "rtems")
  set(soem_rsl_LIB_INSTALL_DIR ${LIB_DIR})
  set(BUILD_TESTS FALSE)
endif()

message(STATUS "OS is ${OS}")

# -------------------------
# Install layout
# -------------------------
set(soem_rsl_INCLUDE_INSTALL_DIR include/soem_rsl)
set(soem_rsl_LIB_INSTALL_DIR lib)

# -------------------------
# Sources / headers (kept)
# -------------------------
file(GLOB soem_rsl_SOURCES soem_rsl/soem_rsl/*.c)
file(GLOB OSAL_SOURCES soem_rsl/osal/${OS}/*.c)
file(GLOB OSHW_SOURCES soem_rsl/oshw/${OS}/*.c)

file(GLOB soem_rsl_HEADERS soem_rsl/soem_rsl/*.h)
file(GLOB OSAL_HEADERS soem_rsl/osal/osal.h soem_rsl/osal/${OS}/*.h)
file(GLOB OSHW_HEADERS soem_rsl/oshw/${OS}/*.h)

add_library(soem_rsl STATIC
  ${soem_rsl_SOURCES}
  ${OSAL_SOURCES}
  ${OSHW_SOURCES}
  ${OSHW_EXTRA_SOURCES}
)

target_link_libraries(soem_rsl ${OS_LIBS})
set_target_properties(soem_rsl PROPERTIES POSITION_INDEPENDENT_CODE ON)

# -------------------------
# Include dirs
#
# IMPORTANT FIX:
# - install interface MUST be "include" (the root),
#   because headers are installed to: include/soem_rsl/*.h
#   so users can do: #include <soem_rsl/ethercat.h>
# -------------------------
target_include_directories(soem_rsl PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/soem_rsl>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/soem_rsl/soem_rsl>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/soem_rsl/osal>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/soem_rsl/osal/${OS}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/soem_rsl/oshw/${OS}>
  $<INSTALL_INTERFACE:include>
)

message(STATUS "LIB_DIR: ${soem_rsl_LIB_INSTALL_DIR}")

# -------------------------
# Install / Export (ament)
# -------------------------
install(TARGETS soem_rsl
  EXPORT export_${PROJECT_NAME}
  ARCHIVE DESTINATION ${soem_rsl_LIB_INSTALL_DIR}
  LIBRARY DESTINATION ${soem_rsl_LIB_INSTALL_DIR}
  RUNTIME DESTINATION bin
)

install(EXPORT export_${PROJECT_NAME}
  DESTINATION share/${PROJECT_NAME}/cmake
)

# Install headers (keep original behavior: flattened into include/soem_rsl)
install(FILES
  ${soem_rsl_HEADERS}
  ${OSAL_HEADERS}
  ${OSHW_HEADERS}
  DESTINATION ${soem_rsl_INCLUDE_INSTALL_DIR}
)

# Keep compatibility: also install ethercattype.h into global include/
install(FILES
  soem_rsl/soem_rsl/ethercattype.h
  DESTINATION include
)

# Export to downstream packages
ament_export_targets(export_${PROJECT_NAME} HAS_LIBRARY_TARGET)
ament_export_include_directories(include)
ament_package()

# -------------------------
# Optional: build SOEM test tools (off by default)
# -------------------------
option(SOEM_RSL_BUILD_TESTS "Build SOEM test tools" OFF)
if(SOEM_RSL_BUILD_TESTS AND (UNIX AND NOT APPLE) AND OS STREQUAL "linux")
  add_subdirectory(soem_rsl/test/linux/slaveinfo)
  add_subdirectory(soem_rsl/test/linux/eepromtool)
  add_subdirectory(soem_rsl/test/linux/simple_test)
endif()
