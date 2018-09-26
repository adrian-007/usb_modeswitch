cmake_minimum_required(VERSION 3.0)
project(usb-modeswitch VERSION 2.5.2)

include(GNUInstallDirs)

# function that will embed Tcl script in C-compatible header file
function(embed_script SCRIPT_NAME HEADER_NAME)
    file(READ ${SCRIPT_NAME} SCRIPT_SRC)

    # split file into lines
    string(REGEX REPLACE ";" "\\\\;" SCRIPT_LINES "${SCRIPT_SRC}")
    string(REGEX REPLACE "\r?\n" ";" SCRIPT_LINES "${SCRIPT_LINES}")

    set(SCRIPT_SRC "")
    foreach(SCRIPT_LINE ${SCRIPT_LINES})
        # replace comments
        string(REGEX REPLACE "^[ \t]*#.*" "" SCRIPT_LINE "${SCRIPT_LINE}")

        # trim leading whitespace
        string(REGEX REPLACE "^[ \t\r\n]+" "" SCRIPT_LINE "${SCRIPT_LINE}")

        # escape \
        string(REPLACE "\\" "\\\\" SCRIPT_LINE "${SCRIPT_LINE}")

        # escape "
        string(REPLACE "\"" "\\\"" SCRIPT_LINE "${SCRIPT_LINE}")

        # escape semicolon due to CMake treating it as a special character
        string(REGEX REPLACE ";" "\\\\;" SCRIPT_LINE "${SCRIPT_LINE}")

        string(LENGTH "${SCRIPT_LINE}" STR_LEN)

        if (${STR_LEN})
            string(CONCAT SCRIPT_SRC "${SCRIPT_SRC}" "${SCRIPT_LINE}" "\\n")
        endif()
    endforeach()

    string(CONCAT SCRIPT_SRC "#define RAW \"" ${SCRIPT_SRC} "\"")

    file(WRITE ${HEADER_NAME} "${SCRIPT_SRC}")
endfunction()

# find usb_modeswitch dependencies
find_path(libusb_INCLUDES NAMES libusb.h PATH_SUFFIXES include libusb libusb-1.0)
find_library(libusb_LIBS NAMES usb-1.0 usb)

if (NOT libusb_LIBS)
    message(FATAL_ERROR Cannot find libusb)
endif()

set(USB_MODESWITCH_TARGET usb_modeswitch)
set(USB_MODESWITCH_SRC
    src/usb_modeswitch.c
)

# usb_modeswitch executable
add_executable(${USB_MODESWITCH_TARGET} ${USB_MODESWITCH_SRC})
target_include_directories(${USB_MODESWITCH_TARGET} PRIVATE
    ${libusb_INCLUDES}
    src
)
target_link_libraries(${USB_MODESWITCH_TARGET}
    ${libusb_LIBS}
)

find_program(TCLSH_EXE tclsh PATHS
    ${CMAKE_INSTALL_FULL_SBINDIR}
    ${CMAKE_INSTALL_FULL_BINDIR}
    /usr/bin
    /usr/sbin
    NO_DEFAULT_PATH
)

if (TCLSH_EXE)
    configure_file(assets/usb_modeswitch.tcl.in ${CMAKE_BINARY_DIR}/usb_modeswitch_dispatcher)

    install(PROGRAMS
        ${CMAKE_BINARY_DIR}/usb_modeswitch_dispatcher
        DESTINATION ${CMAKE_INSTALL_SBINDIR}
    )
else()
    set(TCLSH_EXE "/usr/bin/tclsh")
    configure_file(assets/usb_modeswitch.tcl.in assets/usb_modeswitch.tcl)

    # find usb_modeswitch_dispatcher dependencies
    find_path(jimtcl_INCLUDES NAMES jim.h)
    find_library(jimtcl_LIBS NAMES jim)

    if (NOT jimtcl_LIBS)
        message(FATAL_ERROR Cannot find jimtcl)
    endif()

    set(USB_MODESWITCH_DISPATCHER_TARGET usb_modeswitch_dispatcher)
    set(USB_MODESWITCH_DISPATCHER_SRC
        src/dispatcher.c
    )

    embed_script(assets/usb_modeswitch.tcl.in src/usb_modeswitch.string)

    # usb_modeswitch_dispatcher executable
    add_executable(${USB_MODESWITCH_DISPATCHER_TARGET} ${USB_MODESWITCH_DISPATCHER_SRC})
    target_include_directories(${USB_MODESWITCH_DISPATCHER_TARGET} PRIVATE
        assets
        ${jimtcl_INCLUDES}
    )
    target_link_libraries(${USB_MODESWITCH_DISPATCHER_TARGET}
        ${jimtcl_LIBS}
    )

    install(TARGETS ${USB_MODESWITCH_DISPATCHER_TARGET}
        DESTINATION ${CMAKE_INSTALL_SBINDIR}
    )
endif()

install(TARGETS ${USB_MODESWITCH_TARGET}
    DESTINATION ${CMAKE_INSTALL_SBINDIR}
)

install(DIRECTORY
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/usb_modeswitch
)

install(FILES
    assets/usb_modeswitch.conf
    DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}
)

install(PROGRAMS
    assets/usb_modeswitch.sh
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/udev
    RENAME usb_modeswitch
)

install(FILES
    assets/man/usb_modeswitch.1
    assets/man/usb_modeswitch_dispatcher.1
    DESTINATION ${CMAKE_INSTALL_MANDIR}/man1
)

if (EXISTS ${CMAKE_INSTALL_FULL_SYSCONFDIR}/systemd)
    install(FILES
        assets/usb_modeswitch@.service
        DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/systemd/system
    )
elseif(EXISTS ${CMAKE_INSTALL_FULL_SYSCONFDIR}/init AND EXISTS ${CMAKE_INSTALL_FULL_SBINDIR}/init)
    install(FILES
        assets/usb-modeswitch-upstart.conf
        DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/init
    )
endif()
