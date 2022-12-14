cmake_minimum_required(VERSION 3.15)
project(hwloc
    LANGUAGES C
    VERSION 2.6.0)

# Available configuration options
# HWLOC_SKIP_LSTOPO doesn't build/install lstopo
# HWLOC_SKIP_TOOLS doesn't build/install other hwloc tools
# HWLOC_SKIP_INCLUDES doesn't install headers

set(TOPDIR ../..)

# Configure based on the MSVC config

configure_file(${TOPDIR}/contrib/windows/hwloc_config.h include/hwloc/autogen/config.h COPYONLY)
configure_file(${TOPDIR}/contrib/windows/static-components.h include/static-components.h COPYONLY)
configure_file(${TOPDIR}/contrib/windows/private_config.h include/private/autogen/config.h COPYONLY)

file(READ ${CMAKE_CURRENT_BINARY_DIR}/include/private/autogen/config.h PRIVATE_CONFIG_H)
string(REPLACE "#define SIZEOF_VOID_P 8"        "#define SIZEOF_VOID_P ${CMAKE_SIZEOF_VOID_P}"        PRIVATE_CONFIG_H "${PRIVATE_CONFIG_H}")

# disable x86 entirely by default
string(REPLACE "#define HWLOC_X86_32_ARCH 1" "/* #undef HWLOC_X86_32_ARCH */" PRIVATE_CONFIG_H "${PRIVATE_CONFIG_H}")
string(REPLACE "#define HWLOC_X86_64_ARCH 1" "/* #undef HWLOC_X86_64_ARCH */" PRIVATE_CONFIG_H "${PRIVATE_CONFIG_H}")
# and now reenable x86-36 or x86-64 if detected
if (CMAKE_SYSTEM_PROCESSOR STREQUAL "AMD64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
    # "AMD64" on Windows, "x86_64" on Linux
    string(REPLACE "/* #undef HWLOC_X86_64_ARCH */" "#define HWLOC_X86_64_ARCH 1" PRIVATE_CONFIG_H "${PRIVATE_CONFIG_H}")
elseif (CMAKE_SYSTEM_PROCESSOR STREQUAL "x86" OR CMAKE_SYSTEM_PROCESSOR MATCHES "i.86")
    # "x86" on Windows, "i.86" on Linux
    string(REPLACE "/* #undef HWLOC_X86_32_ARCH */" "#define HWLOC_X86_32_ARCH 1" PRIVATE_CONFIG_H "${PRIVATE_CONFIG_H}")
endif()

# the following lines are disabled until we are sure they are safe with old build environmentx
# - snprintf() returned broken values in the past, hwloc detects it during configure (see 7a4ee26510c06b55fc04aaccbfa18d0ca3b87198)
#   string(REPLACE "#define HAVE_DECL_SNPRINTF 0" "#define HAVE_DECL_SNPRINTF 1" PRIVATE_CONFIG_H "${PRIVATE_CONFIG_H}")
# - strtoull() had some issues in the past (see 9559bd08b79ef63dce45df87fb7f875b73ecb512)
#   string(REPLACE "#define HAVE_DECL_STRTOULL 0" "#define HAVE_DECL_STRTOULL 1" PRIVATE_CONFIG_H "${PRIVATE_CONFIG_H}")

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/include/private/autogen/config.h "${PRIVATE_CONFIG_H}")

# Library

add_compile_options("$<$<CONFIG:DEBUG>:-DHWLOC_DEBUG=1>")

# FIXME dll soname
add_library(libhwloc
    ${TOPDIR}/hwloc/topology.c
    ${TOPDIR}/hwloc/traversal.c
    ${TOPDIR}/hwloc/distances.c
    ${TOPDIR}/hwloc/memattrs.c
    ${TOPDIR}/hwloc/cpukinds.c
    ${TOPDIR}/hwloc/components.c
    ${TOPDIR}/hwloc/bind.c
    ${TOPDIR}/hwloc/bitmap.c
    ${TOPDIR}/hwloc/pci-common.c
    ${TOPDIR}/hwloc/diff.c
    ${TOPDIR}/hwloc/shmem.c
    ${TOPDIR}/hwloc/misc.c
    ${TOPDIR}/hwloc/base64.c
    ${TOPDIR}/hwloc/topology-noos.c
    ${TOPDIR}/hwloc/topology-synthetic.c
    ${TOPDIR}/hwloc/topology-xml.c
    ${TOPDIR}/hwloc/topology-xml-nolibxml.c
    ${TOPDIR}/hwloc/topology-windows.c
    ${TOPDIR}/hwloc/topology-x86.c
)

set_target_properties(libhwloc PROPERTIES DEFINE_SYMBOL _USRDLL)

target_include_directories(libhwloc PRIVATE ${TOPDIR}/include ${TOPDIR}/hwloc ${CMAKE_CURRENT_BINARY_DIR}/include)

# Tools under utils/hwloc

if(NOT HWLOC_SKIP_TOOLS)

    set(TOOLS
        hwloc-bind
        hwloc-calc
        hwloc-diff
        hwloc-distrib
        hwloc-gather-cpuid
        hwloc-info
        hwloc-patch
    )

    foreach(tool IN ITEMS ${TOOLS})
        add_executable(${tool}
            ${TOPDIR}/utils/hwloc/${tool}.c)
        target_include_directories(${tool} PRIVATE ${TOPDIR}/include ${CMAKE_CURRENT_BINARY_DIR}/include)
        target_link_libraries(${tool} PRIVATE libhwloc)
    endforeach(tool)

endif()

# lstopo

if(NOT HWLOC_SKIP_LSTOPO)

    set(LSTOPOS
        lstopo-no-graphics
        lstopo
        lstopo-win
    )

    set(LSTOPO_COMMON_SOURCES
        ${TOPDIR}/utils/lstopo/lstopo.c
        ${TOPDIR}/utils/lstopo/lstopo-draw.c
        ${TOPDIR}/utils/lstopo/lstopo-tikz.c
        ${TOPDIR}/utils/lstopo/lstopo-fig.c
        ${TOPDIR}/utils/lstopo/lstopo-svg.c
        ${TOPDIR}/utils/lstopo/lstopo-ascii.c
        ${TOPDIR}/utils/lstopo/lstopo-text.c
        ${TOPDIR}/utils/lstopo/lstopo-xml.c
        ${TOPDIR}/utils/hwloc/common-ps.c
    )

    add_executable(lstopo-no-graphics
        ${LSTOPO_COMMON_SOURCES}
    )
    target_link_libraries(lstopo-no-graphics PRIVATE libhwloc)

    add_executable(lstopo
        ${LSTOPO_COMMON_SOURCES}
        ${TOPDIR}/utils/lstopo/lstopo-windows.c
    )
    set_target_properties(lstopo PROPERTIES COMPILE_FLAGS "-DLSTOPO_HAVE_GRAPHICS")

    add_executable(lstopo-win
        ${LSTOPO_COMMON_SOURCES}
        ${TOPDIR}/utils/lstopo/lstopo-windows.c
    )
    set_target_properties(lstopo-win PROPERTIES COMPILE_FLAGS "-DLSTOPO_HAVE_GRAPHICS")
    target_link_options(lstopo-win PRIVATE "/subsystem:windows" "/entry:mainCRTStartup")

    foreach(tool IN ITEMS ${LSTOPOS})
        target_include_directories(${tool} PRIVATE ${TOPDIR}/include ${CMAKE_CURRENT_BINARY_DIR}/include ${TOPDIR}/utils/hwloc)
        target_link_libraries(${tool} PRIVATE libhwloc)
    endforeach(tool)

endif()

# Misc

foreach(target IN ITEMS libhwloc ${TOOLS} ${LSTOPOS})
    target_compile_definitions(${target} PRIVATE _CRT_SECURE_NO_WARNINGS)
endforeach(target)

# Install

install(TARGETS libhwloc
    RUNTIME DESTINATION bin
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib)

if(NOT HWLOC_SKIP_TOOLS)
    install(TARGETS ${TOOLS}
        RUNTIME DESTINATION bin)
endif()

if(NOT HWLOC_SKIP_LSTOPO)
    install(TARGETS ${LSTOPOS}
        RUNTIME DESTINATION bin)
endif()

if(NOT HWLOC_SKIP_INCLUDES)
    install(FILES ${TOPDIR}/include/hwloc.h DESTINATION include)
    install(DIRECTORY ${TOPDIR}/include/hwloc DESTINATION include FILES_MATCHING PATTERN "*.h")
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/include/hwloc/autogen/config.h DESTINATION include/hwloc/autogen)
endif()
