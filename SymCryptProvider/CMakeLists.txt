cmake_minimum_required(VERSION 3.13.0)

project(scossl_provider)

set(DEFAULT_BUILD_TYPE "Release")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Wno-unknown-pragmas")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++1y -Wall -Wextra")

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/inc/p_scossl_base.h.in
    ${CMAKE_CURRENT_SOURCE_DIR}/inc/p_scossl_base.h)

find_package(OpenSSL REQUIRED)
include_directories(${OPENSSL_INCLUDE_DIR})

find_library(SYMCRYPT_LIBRARY symcrypt PATHS ${CMAKE_SOURCE_DIR})

set(SCOSSL_SOURCES
    ./src/asymcipher/p_scossl_rsa_cipher.c
    ./src/ciphers/p_scossl_aes.c
    ./src/ciphers/p_scossl_aes_aead.c
    ./src/ciphers/p_scossl_aes_xts.c
    ./src/decoder/p_scossl_decode_common.c
    ./src/decoder/p_scossl_decode_mlkem.c
    ./src/digests/p_scossl_digest_common.c
    ./src/digests/p_scossl_digest_generic.c
    ./src/digests/p_scossl_shake.c
    ./src/digests/p_scossl_cshake.c
    ./src/encoder/p_scossl_encode_common.c
    ./src/encoder/p_scossl_encode_mlkem.c
    ./src/kdf/p_scossl_hkdf.c
    ./src/kdf/p_scossl_kbkdf.c
    ./src/kdf/p_scossl_pbkdf2.c
    ./src/kdf/p_scossl_srtpkdf.c
    ./src/kdf/p_scossl_sshkdf.c
    ./src/kdf/p_scossl_sskdf.c
    ./src/kdf/p_scossl_tls1prf.c
    ./src/kem/p_scossl_mlkem.c
    ./src/keyexch/p_scossl_dh.c
    ./src/keyexch/p_scossl_ecdh.c
    ./src/keyexch/p_scossl_kdf_keyexch.c
    ./src/keymgmt/p_scossl_dh_keymgmt.c
    ./src/keymgmt/p_scossl_ecc_keymgmt.c
    ./src/keymgmt/p_scossl_kdf_keymgmt.c
    ./src/keymgmt/p_scossl_mlkem_keymgmt.c
    ./src/keymgmt/p_scossl_rsa_keymgmt.c
    ./src/mac/p_scossl_cmac.c
    ./src/mac/p_scossl_hmac.c
    ./src/mac/p_scossl_kmac.c
    ./src/signature/p_scossl_ecdsa_signature.c
    ./src/signature/p_scossl_rsa_signature.c
    ./src/p_scossl_bio.c
    ./src/p_scossl_ecc.c
    ./src/p_scossl_rand.c
    ./src/p_scossl_rsa.c
    ./src/p_scossl_base.c
)

if (KEYSINUSE_ENABLED)
    list(APPEND SCOSSL_SOURCES ../KeysInUse/keysinuse.c)
endif()

add_library(scossl_provider SHARED ${SCOSSL_SOURCES})
add_dependencies(scossl_provider scossl_common)

set_target_properties(scossl_provider PROPERTIES PUBLIC_HEADER ./inc/p_scossl_base.h)
target_include_directories(scossl_provider PUBLIC ./inc)
target_include_directories(scossl_provider PRIVATE ./src)
# target_include_directories(scossl_provider PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_include_directories(scossl_provider PUBLIC ../ScosslCommon/inc)

if (KEYSINUSE_ENABLED)
    if (KEYSINUSE_LOG_SYSLOG)
        target_compile_definitions(scossl_provider PRIVATE KEYSINUSE_LOG_SYSLOG)
        target_link_libraries(scossl_provider PUBLIC ${SYSTEMD_LIBRARY})
    endif()

    target_compile_definitions(scossl_provider PRIVATE KEYSINUSE_ENABLED)
    target_include_directories(scossl_provider PUBLIC ../KeysInUse)
endif()

# Remove default "lib" prefix from symcryptprovider.so as OpenSSL provider is not a generic Linux .so
set_target_properties(scossl_provider PROPERTIES PREFIX "")
set_target_properties(scossl_provider PROPERTIES OUTPUT_NAME "symcryptprovider")

target_link_libraries(scossl_provider PRIVATE scossl_common)
target_link_libraries(scossl_provider PUBLIC ${SYMCRYPT_LIBRARY})
target_link_libraries(scossl_provider PUBLIC ${OPENSSL_CRYPTO_LIBRARY})

if (NOT DEFINED OPENSSL_PROVIDERS)
set(OPENSSL_PROVIDERS "${CMAKE_INSTALL_LIBDIR}/ossl-modules" CACHE PATH "Path to OpenSSL providers")
endif()

# Install the engine to the OpenSSL modules directory
# NB: this won't work if the distro has a custom modules directory that doesn't match
# the OpenSSL default.
install(
    TARGETS scossl_provider
    LIBRARY DESTINATION "${OPENSSL_PROVIDERS}"
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
