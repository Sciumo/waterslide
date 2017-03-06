project ( Waterslide C CXX )
cmake_minimum_required ( VERSION 3.1 )

## Waterslide build version
set ( WS_VERSION_MAJOR 1 )
set ( WS_VERSION_MINOR 0 )
set ( WS_VERSION_PATCH 1 )
set ( CLIENT_VERSION ${WS_VERSION_MAJOR}.${WS_VERSION_MINOR}.${WS_VERSION_PATCH} )

set ( SUBMODULES ${CMAKE_CURRENT_LIST_DIR}/submodules )

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")
message (STATUS "module path ${CMAKE_MODULE_PATH}")

include ( FindProtobuf )

find_package ( Threads REQUIRED )
find_package ( Protobuf REQUIRED )
find_package ( BISON REQUIRED )
find_package ( FLEX REQUIRED )
find_package ( ZLIB REQUIRED )
find_package ( HiRedis )
find_package ( Paho )

set ( CMAKE_C_STANDARD 99 )

if ( UNIX )
  add_definitions ( -D__STDC_FORMAT_MACROS )
endif ( UNIX )

set ( CMAKE_C_STANDARD 99 )

message ( STATUS "Build Type ${CMAKE_BUILD_TYPE}" )
option ( DEBUG "Debug build" OFF )
if ( DEBUG )
  add_definitions ( -DDEBUG )
  set ( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g2" )
  set ( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g2" )
  message ( STATUS "Debug info build" )
else ( DEBUG )
  message ( STATUS "No debug info build" )
endif ( DEBUG )

option ( HUGETUPLE "Huge Tuple" OFF )
if ( HUGETUPLE )
  add_definitions ( -DHUGETUPLE )
endif ( HUGETUPLE )

include_directories ( SYSTEM src src/include src/util src/procs/protobuf src/lib )
include_directories ( ${PROTOBUF_INCLUDE_DIRS} )
include_directories ( ${CMAKE_CURRENT_BINARY_DIR} )
include_directories ( ${ZLIB_INCLUDE_DIRS} )

protobuf_generate_cpp ( PROTO_SERIAL_SRCS  PROTO_HDRS src/procs/protobuf/wsserial.proto )
protobuf_generate_cpp ( PROTO_SRCS  PROTO_HDRS src/procs/protobuf/wsproto.proto )

set ( WS_PROTO ${PROTO_SRCS} ${PROTO_SERIAL_SRCS} )
set ( WS_PROTO_LIB ${PROTOBUF_LIBRARIES} ${ZLIB_LIBRARIES} )

FLEX_TARGET ( GraphLex src/lib/graph.l ${CMAKE_CURRENT_BINARY_DIR}/graph.lex.cc COMPILE_FLAGS "-Ppg" )
BISON_TARGET ( GraphTab src/lib/graph.y ${CMAKE_CURRENT_BINARY_DIR}/graph.tab.cc COMPILE_FLAGS "-d -ppg" )

FLEX_TARGET ( WSCalcLex src/lib/wscalc.l ${CMAKE_CURRENT_BINARY_DIR}/wscalc.lex.cc COMPILE_FLAGS "-Pwscalc" )
BISON_TARGET ( WSCalcTab src/lib/wscalc.y ${CMAKE_CURRENT_BINARY_DIR}/wscalc.tab.cc COMPILE_FLAGS "-d -pwscalc" )

set (WSLIBSRC
  src/lib/ahocorasick.c
  src/lib/ast.cc
  src/lib/fileout.c
  src/lib/init.c
  src/lib/label_match.c
  src/lib/listhash.c
  src/lib/graphBuilder.cc
  src/lib/mimo.c
  src/procs/proc_calc.c
  src/lib/setup_exit.c
  src/lib/sht_expire_cnt.c
  src/lib/sht_registry.c
  src/lib/so_loader.c
  src/lib/stringmatch.c
  src/lib/sysutil.c
  src/lib/variance.c
  src/lib/waterslide_io.c
  src/lib/waterslidedata.c
  src/lib/wsprocbuffer.c
  src/lib/wsprocess.c
  src/lib/wsprockeystate.c
  src/lib/wstypes.c
  ${BISON_GraphTab_OUTPUTS} ${FLEX_GraphLex_OUTPUTS}  ${BISON_WSCalcTab_OUTPUTS} ${FLEX_WSCalcLex_OUTPUTS} )
set ( WSSRC src/util/waterslide.c )

add_library ( wslib SHARED ${WSLIBSRC} )
target_link_libraries ( wslib ${PROTOBUF_LIBRARIES} ${ZLIB_LIBRARIES} )
set_property ( TARGET wslib PROPERTY CXX_STANDARD 11 )

add_executable ( ws ${WSSRC} )
target_link_libraries ( ws ${PROTOBUF_LIBRARIES} ${ZLIB_LIBRARIES} wslib ${CMAKE_DL_LIBS} )
set_property ( TARGET ws PROPERTY CXX_STANDARD 11 )

add_executable ( wsman ./src/util/wsman.c ./src/lib/init.c )
target_link_libraries ( wsman wslib ${CMAKE_DL_LIBS} )

add_executable ( wsalias ./src/util/wsalias.c ./src/lib/init.c )
target_link_libraries ( wsalias wslib ${CMAKE_DL_LIBS} )

set_target_properties ( ws wslib wsman wsalias
    PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/../bin"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/../bin"
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/../bin"
)

function( Kid name file extrasrc extralib )
  message ( STATUS "Configure Kid ${name} ${file}" )
  add_library ( ${name} SHARED ${file} ${extrasrc} )
  target_link_libraries ( ${name} ${PROTOBUF_LIBRARIES} ${ZLIB_LIBRARIES} wslib ${extralib} )
  set_target_properties ( ${name}
      PROPERTIES
      ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/../procs"
      LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/../procs"
      RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/../procs"
      PREFIX "proc_"
      SUFFIX ".ws_so"
  )
endfunction(Kid)

function(Kids)
  foreach( KID ${ARGN} )
      Kid ( ${KID} "src/procs/proc_${KID}.c" "" "" )
  endforeach(KID)
endfunction()

if ( HIREDIS_FOUND )
  Kids ( redis )
endif ( HIREDIS_FOUND )

Kid ( proto_in "src/procs/proc_wsproto_in.cc" "${WS_PROTO}" "${WSPROTO_LIB}" )
Kid ( proto_out "src/procs/proc_wsproto_out.cc" "${WS_PROTO}" "${WSPROTO_LIB}" )

#todo add RE2
#Kid ( re2 "src/procs/proc_re2.cc" ${WS_PROTO} "" )

Kids (
  addlabelmember
  appendfirstitem
  appendlast
  asciihex
  bandwidth
  base64
  bloom
  bundle
  calc
  charcnt
  cntquery
  combinestrings
  countlabels
  csv_in
  data2label
  decodejson
  denest
  dupestring
  dupetuple
  encodebase64
  equal
  exactmatch
  exec_in
  file_in
#  filemagic  #missing magic.h
  fillmissing
  firstn
#  fixedmatch #compile issue
  flush
  haslabel
  heavyhitters
  jointuple
  keeplast
  keepn
  keyadd_initial_custom
  keyadd_initial
  keyadd
  keyaverage
  keycount
  keyflow
  keyrate
  keytime
  keyvariance
  label
  labelmultiple
  lastn
  loadbalance
  match
  match_uint
  mergetuple
  metadice
  mklabelset
  mknest
  print
  print2json
  rbsort
  sed
  sort
  splitstring
  top
  uniq
)

option( DOCS "Latex documents build" OFF )
if ( DOCS )
  message ( STATUS "Latex documents build" )
  find_package ( LATEX )
  if ( LATEX_FOUND )
    include ( UseLatex )
    add_latex_document ( doc/users_guide/waterslide_users_guide.tex FORCE_PDF )
    add_latex_document ( doc/developer/waterslide_programming_guide.tex FORCE_PDF )
    add_latex_document ( doc/developer/waterslide_refsheet.tex FORCE_PDF )
  else ( LATEX_FOUND )
    message ( STATUS "Documents not built. No latex found." )
  endif ( LATEX_FOUND )
else ( DOCS )
  message ( STATUS "No Latex documents build" )
endif ( DOCS )
