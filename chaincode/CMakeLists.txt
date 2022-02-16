cmake_minimum_required(VERSION 3.5.1)

set(SOURCE_FILES
    gpm.cpp
    ../lib/pdid_gpm.cpp
    ../lib/pdid.c
    ../lib/utils.c
    ../lib/tweetnacl.c
    ../lib/randombytes.c
    ../lib/ecc.c
    )
    include(${CMAKE_CURRENT_LIST_DIR}/../../cmake/Init.cmake)
    include(${CMAKE_CURRENT_LIST_DIR}/../../cmake/ConfigSGX.cmake)
    
    set(COMMON_SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/../../common)
    set(ECC_ENCLAVE_DIR ${CMAKE_CURRENT_LIST_DIR}/../../ecc_enclave)
    
    include(${ECC_ENCLAVE_DIR}/enclave/CMakeVariables.txt)
    
    set(cleanup_files
        )
    
    set_directory_properties(PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES "${cleanup_files}")
    
    add_library(enclave SHARED ${SOURCE_FILES})
    
    include_directories(
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${CMAKE_CURRENT_SOURCE_DIR}/../lib
        ${COMMON_SOURCE_DIR}
        ${COMMON_SOURCE_DIR}/json
        ${ECC_ENCLAVE_DIR}/enclave
        ${SGX_SDK}/include
        ${SGX_SDK}/include/tlibc
        ${SGX_SDK}/include/libcxx
        ${COMMON_SOURCE_DIR}/base64
        ${COMMON_SOURCE_DIR}/logging/trusted
        ${SGX_SSL}/include
        ${COMMON_SOURCE_DIR}/crypto/pdo/common
        )
    
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${SGX_COMMON_CFLAGS} -nostdinc -fno-builtin -fvisibility=hidden -fpie -fstack-protector -std=c11")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${SGX_COMMON_CFLAGS} -nostdinc -fno-builtin -fvisibility=hidden -fpie -fstack-protector -std=c++11 -nostdinc++ -fpermissive")
    
    add_definitions(-DENCLAVE_CODE -DWITH_SGX)

    set_target_properties(enclave PROPERTIES PREFIX "")

    target_link_libraries(enclave "${SGX_COMMON_CFLAGS} -Wl,--no-undefined -nostdlib -nodefaultlibs -nostartfiles \
    -L${COMMON_SOURCE_DIR}/logging/_build \
    -L${COMMON_SOURCE_DIR}/crypto/_build \
    -L${ECC_ENCLAVE_DIR}/_build/lib \
    -Wl,--start-group -l${T_CRYPTO_LIB_NAME} -l${SHIM_LIB_NAME} -l${T_CRYPTO_ADAPT_LIB_NAME} -l${T_LOGGING_LIB_NAME} -Wl,--end-group \
    -L${SGX_SSL_LIBRARY_PATH} \
    -Wl,--whole-archive -lsgx_tsgxssl -Wl,--no-whole-archive -lsgx_tsgxssl_crypto \
    -L${SGX_LIBRARY_PATH} \
    -Wl,--whole-archive -l${SGX_TRTS_LIB} -Wl,--no-whole-archive \
    -Wl,--start-group -lsgx_tstdc -lsgx_tcxx -lsgx_tcrypto -l${SGX_TSVC_LIB} -Wl,--end-group \
    -Wl,-Bstatic -Wl,-Bsymbolic -Wl,--no-undefined \
    -Wl,-pie,-eenclave_entry -Wl,--export-dynamic \
    -Wl,--defsym,__ImageBase=0 \
    -Wl,--version-script=${ECC_ENCLAVE_DIR}/enclave/enclave.lds")
    
    add_custom_command(TARGET enclave
        POST_BUILD
        COMMENT "Create enclave signer's key"
        COMMAND openssl genrsa -3 -out ${CMAKE_BINARY_DIR}/enclave_signer_private.pem 3072
        COMMENT "Signing the enclave => ${CMAKE_BINARY_DIR}/lib/enclave.signed.so"
        COMMAND mkdir -p ${CMAKE_BINARY_DIR}/lib
        COMMAND ${SGX_ENCLAVE_SIGNER} sign
        -key ${CMAKE_BINARY_DIR}/enclave_signer_private.pem
        -config ${ECC_ENCLAVE_DIR}/enclave/enclave.config.xml
        -enclave enclave.so
        -out ${CMAKE_BINARY_DIR}/lib/enclave.signed.so
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR})
    
    add_custom_command(TARGET enclave
        POST_BUILD
        COMMAND ${COMMON_SOURCE_DIR}/enclave/generate_mrenclave.sh ${CMAKE_BINARY_DIR} ${ECC_ENCLAVE_DIR}/enclave
        COMMAND mv ${CMAKE_BINARY_DIR}/mrenclave ${CMAKE_BINARY_DIR}/lib
        COMMAND mv ${CMAKE_BINARY_DIR}/mrenclave.go ${CMAKE_BINARY_DIR}/lib
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        COMMENT "Create enclave mrenclave"
        )
