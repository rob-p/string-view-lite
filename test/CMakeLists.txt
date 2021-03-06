# Copyright 2017-2018 by Martin Moene
#
# https://github.com/martinmoene/string-view-lite
#
# Distributed under the Boost Software License, Version 1.0.
# (See accompanying file LICENSE.txt or copy at http://www.boost.org/LICENSE_1_0.txt)

cmake_minimum_required( VERSION 3.5 )

project( test LANGUAGES CXX )

set( SOURCES string-view-lite.t.cpp string-view.t.cpp )
set( HDRNAME string_view.hpp )

set( HDRDIR  ${PROJECT_SOURCE_DIR}/../include/nonstd )
set( HDRPATH ${HDRDIR}/${HDRNAME} )

# option text to Boolean:

macro( opt option_text var )
    if( ${option_text} )
        set( ${var} 1 )
    else()
        set( ${var} 0 )
    endif()
endmacro()

opt( STRINGVIEW_LITE_OPT_CONV_STD_STRING_CLASS_METHODS  nssv_CONFIG_CONVERSION_STD_STRING_CLASS_METHODS  )
opt( STRINGVIEW_LITE_OPT_CONV_STD_STRING_FREE_FUNCTIONS nssv_CONFIG_CONVERSION_STD_STRING_FREE_FUNCTIONS )

set( DEFCMN  -Dnssv_STRING_VIEW_HEADER=\"${HDRNAME}\"
             -Dnssv_CONFIG_CONVERSION_STD_STRING_CLASS_METHODS=${nssv_CONFIG_CONVERSION_STD_STRING_CLASS_METHODS}
             -Dnssv_CONFIG_CONVERSION_STD_STRING_FREE_FUNCTIONS=${nssv_CONFIG_CONVERSION_STD_STRING_FREE_FUNCTIONS} )

set( OPTIONS     "" )
set( DEFINITIONS ${DEFCMN} )

set( HAS_STD_FLAGS  FALSE )
set( HAS_CPP98_FLAG FALSE )
set( HAS_CPP11_FLAG FALSE )
set( HAS_CPP14_FLAG FALSE )
set( HAS_CPP17_FLAG FALSE )
set( HAS_CPP20_FLAG FALSE )
set( HAS_CPPLATEST_FLAG FALSE )

if( MSVC )
    set( HAS_STD_FLAGS TRUE )

    set( OPTIONS     -W3 -EHsc )
    set( DEFINITIONS -D_SCL_SECURE_NO_WARNINGS -DNOMINMAX ${DEFCMN} )

    if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 19.00 )
        set( HAS_CPP14_FLAG TRUE )
        set( HAS_CPPLATEST_FLAG TRUE )
    endif()
    if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 19.11 )
        set( HAS_CPP17_FLAG TRUE )
    endif()

elseif( CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang" )

    set( HAS_STD_FLAGS  TRUE )
    set( HAS_CPP98_FLAG TRUE )

    set( OPTIONS     -Wall -Wno-missing-braces -Wno-unnamed-type-template-args -fno-elide-constructors )
    set( DEFINITIONS ${DEFCMN} )

    # GNU: available -std flags depends on version
    if( CMAKE_CXX_COMPILER_ID MATCHES "GNU" )
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.8.0 )
            set( HAS_CPP11_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9.2 )
            set( HAS_CPP14_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 7.1.0 )
            set( HAS_CPP17_FLAG TRUE )
        endif()
    endif()

    # Clang: available -std flags depends on version
    if( CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 3.3.0 )
            set( HAS_CPP11_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 3.4.0 )
            set( HAS_CPP14_FLAG TRUE )
        endif()
        if( NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.0.0 )
            set( HAS_CPP17_FLAG TRUE )
        endif()
    endif()

elseif( CMAKE_CXX_COMPILER_ID MATCHES "Intel" )
# as is
else()
# as is
endif()

# enable MS C++ Core Guidelines checker if MSVC:

function( enable_msvs_guideline_checker target )
    if( MSVC )
        set_target_properties( ${target} PROPERTIES
        VS_GLOBAL_EnableCppCoreCheck true
        VS_GLOBAL_CodeAnalysisRuleSet CppCoreCheckRules.ruleset
        VS_GLOBAL_RunCodeAnalysis true )
    endif()
endfunction()

# make target, compile for given standard if specified:

function( make_target target std )
    add_executable            ( ${target} ${SOURCES} ${HDRPATH} )
    target_compile_options    ( ${target} PRIVATE ${OPTIONS} )
    target_compile_definitions( ${target} PRIVATE ${DEFINITIONS} )
    target_include_directories( ${target} PRIVATE ${HDRDIR} )
    if( std )
        if( MSVC )
            target_compile_options( ${target} PRIVATE -std:${std} )
        else()
            # Necessary for clang 3.x:
            target_compile_options( ${target} PRIVATE -std=${std} )
            # Ok for clang 4 and later:
            # set( CMAKE_CXX_STANDARD ${std} )
            # set( CMAKE_CXX_STANDARD_REQUIRED ON )
            # set( CMAKE_CXX_EXTENSIONS OFF )
        endif()
    endif()
endfunction()

# add generic executable, unless -std flags can be specified:

if( NOT HAS_STD_FLAGS )
    make_target( string-view-lite.t "" )
else()
    # unconditionally add C++98 variant as MSVC has no option for it:
    if( HAS_CPP98_FLAG )
        make_target( string-view-lite-cpp98.t c++98 )
    else()
        make_target( string-view-lite-cpp98.t "" )
    endif()

    if( HAS_CPP11_FLAG )
        make_target( string-view-lite-cpp11.t c++11 )
    endif()

    if( HAS_CPP14_FLAG )
        make_target( string-view-lite-cpp14.t c++14 )
    endif()

    if( HAS_CPP17_FLAG )
        make_target( string-view-lite-cpp17.t c++17 )
        enable_msvs_guideline_checker( string-view-lite-cpp17.t )
    endif()

    if( HAS_CPPLATEST_FLAG )
        make_target( string-view-lite-cpplatest.t c++latest )
    endif()
endif()

# with C++17, honour explicit request for std::string_view or nonstd::string_view:

if( HAS_CPP17_FLAG )
    if ( STRINGVIEW_LITE_OPT_SELECT_STDSV )
        target_compile_definitions( string-view-lite-cpp17.t PRIVATE nssv_CONFIG_SELECT_STD_STRING_VIEW=1 )
    endif()

    if ( STRINGVIEW_LITE_OPT_SELECT_NONSTDSV )
        target_compile_definitions( string-view-lite-cpp17.t PRIVATE nssv_CONFIG_SELECT_NONSTD_STRING_VIEW=1 )
    endif()
endif()

# configure unit tests via CTest:

enable_testing()

if( HAS_STD_FLAGS )
    # unconditionally add C++98 variant for MSVC:
    add_test(     NAME test-cpp98     COMMAND string-view-lite-cpp98.t )

    if( HAS_CPP11_FLAG )
        add_test( NAME test-cpp11     COMMAND string-view-lite-cpp11.t )
    endif()
    if( HAS_CPP14_FLAG )
        add_test( NAME test-cpp14     COMMAND string-view-lite-cpp14.t )
    endif()
    if( HAS_CPP17_FLAG )
        add_test( NAME test-cpp17     COMMAND string-view-lite-cpp17.t )
    endif()
    if( HAS_CPPLATEST_FLAG )
        add_test( NAME test-cpplatest COMMAND string-view-lite-cpplatest.t )
    endif()
else()
    add_test(     NAME test           COMMAND string-view-lite.t --pass )
    add_test(     NAME list_version   COMMAND string-view-lite.t --version )
    add_test(     NAME list_tags      COMMAND string-view-lite.t --list-tags )
    add_test(     NAME list_tests     COMMAND string-view-lite.t --list-tests )
endif()

# end of file
