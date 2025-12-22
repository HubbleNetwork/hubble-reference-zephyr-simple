# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.20.0)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(hubble-demo-app)

# Get the Hubble device key
if(DEFINED ENV{HUBBLE_DEVICE_KEY})
  message(STATUS "Using HUBBLE_DEVICE_KEY from environment")
  set(HUBBLE_DEVICE_KEY $ENV{HUBBLE_DEVICE_KEY})
else()
  message(STATUS "HUBBLE_DEVICE_KEY not set in environment; using default test key")
  set(HUBBLE_DEVICE_KEY 1111111111111111111111111111111111111111111=)
endif()

# Get seconds since epoch (UTC). "%s" yields epoch seconds on most systems.
string(TIMESTAMP EPOCH_S "%s" UTC)

# If %s is not supported on the platform, EPOCH_S may be empty.
if(NOT EPOCH_S)
  message(STATUS "CMake string(TIMESTAMP \"%s\") unsupported; falling back to 'date'")
  execute_process(COMMAND date +%s
                  OUTPUT_VARIABLE EPOCH_S
                  OUTPUT_STRIP_TRAILING_WHITESPACE)
endif()

# Convert to milliseconds. Use math(EXPR) or simple concatenation:
math(EXPR TIME_MS "${EPOCH_S} * 1000")

message(STATUS "Build TIME_MS = ${TIME_MS}")
message(STATUS "Build HUBBLE_DEVICE_KEY = ${HUBBLE_DEVICE_KEY}")

target_compile_definitions(app PRIVATE HUBBLE_KEY=\"${HUBBLE_DEVICE_KEY}\")
target_compile_definitions(app PRIVATE TIME=${TIME_MS})
target_sources(app PRIVATE src/main.c)
target_sources(app PRIVATE src/b64.c)
