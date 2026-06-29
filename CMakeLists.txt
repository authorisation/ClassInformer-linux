cmake_minimum_required(VERSION 3.27 FATAL_ERROR)

project(ida_class_informer)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 11)

# Use submodule if env is not overriding it
if(NOT DEFINED ENV{IDASDK})
    set(ENV{IDASDK} "${CMAKE_CURRENT_SOURCE_DIR}/ida-sdk")
    message(STATUS "Using bundled ida-sdk submodule")
endif()

include("${CMAKE_CURRENT_SOURCE_DIR}/ida-sdk/src/cmake/bootstrap.cmake")
find_package(idasdk REQUIRED)

set(IDA_PLUGIN_DIR "${CMAKE_BINARY_DIR}")

# If build_qt was already run, force all Qt cmake dirs to the built Qt
if(EXISTS "${CMAKE_BINARY_DIR}/qt-install/lib/cmake/Qt6")
    set(_QT_CMAKE "${CMAKE_BINARY_DIR}/qt-install/lib/cmake")
    foreach(_pkg Qt6 Qt6Core Qt6CoreTools Qt6Gui Qt6GuiTools Qt6Widgets Qt6WidgetsTools Qt6OpenGL Qt6OpenGLWidgets)
        set(${_pkg}_DIR "${_QT_CMAKE}/${_pkg}" CACHE PATH "" FORCE)
    endforeach()
    unset(_QT_CMAKE)
endif()

find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets)

set(SOURCES
    Main.cpp
    MainDialog.cpp
    RTTI.cpp
    Vftable.cpp
)

qt6_wrap_ui(UI_HEADERS dialog.ui)
qt6_add_resources(QRC_SOURCES ClassInformerRes.qrc OPTIONS --no-zstd)

if(APPLE)
    if(NOT DEFINED IDA_FRAMEWORKS)
        get_filename_component(_ida_bundle_contents "$ENV{IDABIN}/.." ABSOLUTE)
        if(EXISTS "${_ida_bundle_contents}/Frameworks/QtCore.framework")
            set(IDA_FRAMEWORKS "${_ida_bundle_contents}/Frameworks")
        else()
            foreach(_p
                "/Applications/IDA Professional 9.2.app/Contents/Frameworks"
                "/Applications/IDA Professional 9.1.app/Contents/Frameworks"
                "/Applications/IDA Professional.app/Contents/Frameworks")
                if(EXISTS "${_p}/QtCore.framework")
                    set(IDA_FRAMEWORKS "${_p}")
                    break()
                endif()
            endforeach()
        endif()
        if(NOT IDA_FRAMEWORKS)
            message(FATAL_ERROR
                "Cannot find IDA Qt frameworks. Set -DIDA_FRAMEWORKS=<path> "
                "or set IDABIN to IDA.app/Contents/MacOS/")
        endif()
    endif()
    message(STATUS "IDA Qt frameworks: ${IDA_FRAMEWORKS}")

    ida_add_plugin(class_informer
        SOURCES
            ${SOURCES}
            ${QRC_SOURCES}
            ${UI_HEADERS}
        INCLUDES
            "${CMAKE_SOURCE_DIR}"
            "${CMAKE_CURRENT_BINARY_DIR}"
            ${Qt6Core_INCLUDE_DIRS}
            ${Qt6Widgets_INCLUDE_DIRS}
            ${Qt6Gui_INCLUDE_DIRS}
        LIBRARIES
            "${IDA_FRAMEWORKS}/QtCore.framework/Versions/A/QtCore"
            "${IDA_FRAMEWORKS}/QtGui.framework/Versions/A/QtGui"
            "${IDA_FRAMEWORKS}/QtWidgets.framework/Versions/A/QtWidgets"
        DEFINES
            QT_NAMESPACE=QT
            QT_NO_VERSION_TAGGING
    )

    set_target_properties(class_informer PROPERTIES AUTOMOC ON)

    # On macOS, rewrite Qt framework paths and rpaths so IDA's bundled Qt is used at runtime
    add_custom_command(TARGET class_informer POST_BUILD
        COMMAND install_name_tool -change
            "$<TARGET_SONAME_FILE:Qt6::Widgets>"
            "@rpath/QtWidgets.framework/Versions/A/QtWidgets"
            "$<TARGET_FILE:class_informer>"
        COMMAND install_name_tool -change
            "$<TARGET_SONAME_FILE:Qt6::Gui>"
            "@rpath/QtGui.framework/Versions/A/QtGui"
            "$<TARGET_FILE:class_informer>"
        COMMAND install_name_tool -change
            "$<TARGET_SONAME_FILE:Qt6::Core>"
            "@rpath/QtCore.framework/Versions/A/QtCore"
            "$<TARGET_FILE:class_informer>"
        COMMAND install_name_tool -add_rpath
            "@loader_path/../Frameworks"
            "$<TARGET_FILE:class_informer>" || true
        COMMENT "Rewriting Qt framework paths for IDA compatibility"
        VERBATIM
    )

else()
    # On Linux and Windows link directly against Qt6 found by CMake.
    ida_add_plugin(class_informer
        SOURCES
            ${SOURCES}
            ${QRC_SOURCES}
            ${UI_HEADERS}
        INCLUDES
            "${CMAKE_SOURCE_DIR}"
            "${CMAKE_CURRENT_BINARY_DIR}"
        LIBRARIES
            Qt6::Core
            Qt6::Gui
            Qt6::Widgets
        DEFINES
            QT_NAMESPACE=QT
            QT_NO_VERSION_TAGGING
    )

    set_target_properties(class_informer PROPERTIES AUTOMOC ON)

    if(UNIX AND NOT APPLE)
        # Let the plugin find IDAs Qt at runtime via RPATH
        set_target_properties(class_informer PROPERTIES
            INSTALL_RPATH "$ORIGIN:$ORIGIN/.."
            BUILD_WITH_INSTALL_RPATH TRUE
    )
    endif()
endif()
