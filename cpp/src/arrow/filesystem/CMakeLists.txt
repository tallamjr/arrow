# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

# Headers: top level
arrow_install_all_headers("arrow/filesystem")

# pkg-config support
arrow_add_pkg_config("arrow-filesystem")

add_arrow_test(filesystem-test
               SOURCES
               filesystem_test.cc
               localfs_test.cc
               EXTRA_LABELS
               filesystem
               DEFINITIONS
               ARROW_FILESYSTEM_EXAMPLE_LIBPATH="$<TARGET_FILE:arrow_filesystem_example>"
               EXTRA_DEPENDENCIES
               arrow_filesystem_example)

if(ARROW_BUILD_BENCHMARKS)
  add_arrow_benchmark(localfs_benchmark
                      PREFIX
                      "arrow-filesystem"
                      SOURCES
                      localfs_benchmark.cc
                      STATIC_LINK_LIBS
                      ${ARROW_BENCHMARK_LINK_LIBS})
endif()

if(ARROW_GCS)
  add_arrow_test(gcsfs_test
                 EXTRA_LABELS
                 filesystem
                 EXTRA_LINK_LIBS
                 google-cloud-cpp::storage)
endif()

if(ARROW_AZURE)
  add_arrow_test(azurefs_test
                 EXTRA_LABELS
                 filesystem
                 EXTRA_LINK_LIBS
                 ${AZURE_SDK_LINK_LIBRARIES})
endif()

if(ARROW_S3)
  set(ARROW_S3_TEST_EXTRA_LINK_LIBS)
  # arrow_shared/arrow_static is specified implicitly via
  # arrow_testing_shared/arrow_testing_static but we specify
  # arrow_shared/arrow_static explicitly here to ensure using libarrow
  # before libaws* on link. If we use libaws*.a before libarrow,
  # static variables storage of AWS SDK for C++ in libaws*.a may be
  # mixed with one in libarrow.
  if(ARROW_TEST_LINKAGE STREQUAL "shared")
    list(APPEND ARROW_S3_TEST_EXTRA_LINK_LIBS arrow_shared)
  else()
    list(APPEND ARROW_S3_TEST_EXTRA_LINK_LIBS arrow_static)
  endif()
  list(APPEND ARROW_S3_TEST_EXTRA_LINK_LIBS ${AWSSDK_LINK_LIBRARIES})
  add_arrow_test(s3fs_test
                 SOURCES
                 s3fs_test.cc
                 s3_test_util.cc
                 EXTRA_LABELS
                 filesystem
                 EXTRA_LINK_LIBS
                 ${ARROW_S3_TEST_EXTRA_LINK_LIBS})
  if(TARGET arrow-s3fs-test)
    set(ARROW_S3FS_TEST_COMPILE_DEFINITIONS)
    get_target_property(AWS_CPP_SDK_S3_TYPE aws-cpp-sdk-s3 TYPE)
    # We need to initialize AWS SDK for C++ for direct use (not via
    # arrow::fs::S3FileSystem) in arrow-s3fs-test if we use static AWS
    # SDK for C++ and hide symbols of them. Because AWS SDK for C++
    # has internal static variables that aren't shared in libarrow and
    # arrow-s3fs-test. It means that arrow::fs::InitializeS3() doesn't
    # initialize AWS SDK for C++ that is directly used in
    # arrow-s3fs-test.
    if(AWS_CPP_SDK_S3_TYPE STREQUAL "STATIC_LIBRARY"
       AND CXX_LINKER_SUPPORTS_VERSION_SCRIPT)
      list(APPEND ARROW_S3FS_TEST_COMPILE_DEFINITIONS "AWS_CPP_SDK_S3_PRIVATE_STATIC")
    endif()
    target_compile_definitions(arrow-s3fs-test
                               PRIVATE ${ARROW_S3FS_TEST_COMPILE_DEFINITIONS})
  endif()

  if(ARROW_BUILD_TESTS)
    add_executable(arrow-s3fs-narrative-test s3fs_narrative_test.cc)
    target_link_libraries(arrow-s3fs-narrative-test ${ARROW_TEST_LINK_LIBS}
                          ${GFLAGS_LIBRARIES})
    add_dependencies(arrow-tests arrow-s3fs-narrative-test)
  endif()

  if(ARROW_BUILD_BENCHMARKS AND ARROW_PARQUET)
    add_arrow_benchmark(s3fs_benchmark
                        PREFIX
                        "arrow-filesystem"
                        SOURCES
                        s3fs_benchmark.cc
                        s3_test_util.cc
                        STATIC_LINK_LIBS
                        ${AWSSDK_LINK_LIBRARIES}
                        ${ARROW_BENCHMARK_LINK_LIBS})
    if(ARROW_TEST_LINKAGE STREQUAL "static")
      target_link_libraries(arrow-filesystem-s3fs-benchmark PRIVATE parquet_static)
    else()
      target_link_libraries(arrow-filesystem-s3fs-benchmark PRIVATE parquet_shared)
    endif()
  endif()

  if(ARROW_S3_MODULE)
    add_arrow_test(s3fs_module_test
                   SOURCES
                   s3fs_module_test.cc
                   s3_test_util.cc
                   EXTRA_LABELS
                   filesystem
                   DEFINITIONS
                   ARROW_S3_LIBPATH="$<TARGET_FILE:arrow_s3fs>"
                   EXTRA_LINK_LIBS
                   Boost::filesystem
                   Boost::system)
    target_compile_definitions(arrow-filesystem-test
                               PUBLIC ARROW_S3_LIBPATH="$<TARGET_FILE:arrow_s3fs>")
    target_sources(arrow-filesystem-test PUBLIC s3fs_module_test.cc s3_test_util.cc)
    target_link_libraries(arrow-filesystem-test PUBLIC Boost::filesystem Boost::system)
  endif()
endif()

if(ARROW_HDFS)
  add_arrow_test(hdfs_test EXTRA_LABELS filesystem)
endif()
