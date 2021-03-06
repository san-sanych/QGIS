SET (PYTHON_OUTPUT_DIRECTORY ${QGIS_OUTPUT_DIRECTORY}/python)

ADD_SUBDIRECTORY(plugins)
ADD_SUBDIRECTORY(qsci_apis)
ADD_SUBDIRECTORY(console)
ADD_SUBDIRECTORY(pyplugin_installer)

IF (WITH_PYSPATIALITE)
  ADD_SUBDIRECTORY(pyspatialite)
ENDIF (WITH_PYSPATIALITE)

IF (WITH_INTERNAL_SPATIALITE)
  INCLUDE_DIRECTORIES(
    ../src/core/spatialite/headers
    ../src/core/spatialite/headers/spatialite
  )
ENDIF (WITH_INTERNAL_SPATIALITE)

SET (QGIS_PYTHON_OUTPUT_DIRECTORY ${PYTHON_OUTPUT_DIRECTORY}/qgis)
FILE (MAKE_DIRECTORY ${QGIS_PYTHON_OUTPUT_DIRECTORY})
SET (CMAKE_RUNTIME_OUTPUT_DIRECTORY ${QGIS_PYTHON_OUTPUT_DIRECTORY})
SET (CMAKE_LIBRARY_OUTPUT_DIRECTORY ${QGIS_PYTHON_OUTPUT_DIRECTORY})

INCLUDE_DIRECTORIES(
  ${PYTHON_INCLUDE_PATH}
  ${SIP_INCLUDE_DIR}
  ${QT_QTCORE_INCLUDE_DIR}
  ${QT_QTGUI_INCLUDE_DIR}
  ${QT_QTNETWORK_INCLUDE_DIR}
  ${QT_QTSVG_INCLUDE_DIR}
  ${QT_QTXML_INCLUDE_DIR}
  ${GDAL_INCLUDE_DIR}
  ${GEOS_INCLUDE_DIR}
  ${QWT_INCLUDE_DIR}
  ${QEXTSERIALPORT_INCLUDE_DIR}

  ../src/core
  ../src/core/pal
  ../src/core/composer
  ../src/core/diagram
  ../src/core/dxf
  ../src/core/gps
  ../src/core/raster
  ../src/core/symbology-ng

  ../src/gui/raster
  ../src/gui/attributetable
  ../src/gui/editorwidgets
  ../src/gui/editorwidgets/core

  ${CMAKE_BINARY_DIR}  # qgsconfig.h, qgsversion.h
)

IF(NOT ANDROID)
  SET(SIP_DISABLE_FEATURES ${SIP_DISABLE_FEATURES} ANDROID)
ENDIF(NOT ANDROID)

IF(CMAKE_SYSTEM_PROCESSOR MATCHES "^arm")
ELSE(CMAKE_SYSTEM_PROCESSOR MATCHES "^arm")
  SET(SIP_DISABLE_FEATURES ${SIP_DISABLE_FEATURES} ARM)
ENDIF(CMAKE_SYSTEM_PROCESSOR MATCHES "^arm")

IF(NOT WITH_TOUCH)
  SET(SIP_DISABLE_FEATURES ${SIP_DISABLE_FEATURES} HAVE_TOUCH)
ENDIF(NOT WITH_TOUCH)

IF(NOT QT_MOBILITY_LOCATION_FOUND)
  SET(SIP_DISABLE_FEATURES ${SIP_DISABLE_FEATURES} MOBILITY_LOCATION)
ENDIF(NOT QT_MOBILITY_LOCATION_FOUND)

IF(PYQT4_VERSION_NUM LESS 263680)	# 0x040600
  SET(SIP_DISABLE_FEATURES ${SIP_DISABLE_FEATURES} PROXY_FACTORY)
ENDIF(PYQT4_VERSION_NUM LESS 263680)

IF(NOT PYQT4_VERSION_NUM LESS 263941)	# 0x040705
  SET(SIP_DISABLE_FEATURES ${SIP_DISABLE_FEATURES} QSETINT_CONVERSION)
ENDIF(NOT PYQT4_VERSION_NUM LESS 263941)

IF(NOT PYQT4_VERSION_NUM LESS 264194)	# 0x040802
  SET(SIP_DISABLE_FEATURES ${SIP_DISABLE_FEATURES} QSETTYPE_CONVERSION)
ENDIF(NOT PYQT4_VERSION_NUM LESS 264194)

IF(PYQT4_VERSION_NUM LESS 264196)       # 0x040804
  SET(SIP_DISABLE_FEATURES ${SIP_DISABLE_FEATURES} QLISTCONSTPTR_CONVERSION)
ENDIF(PYQT4_VERSION_NUM LESS 264196)

# core module
FILE(GLOB_RECURSE sip_files_core core/*.sip)
SET(SIP_EXTRA_FILES_DEPEND ${sip_files_core})
SET(SIP_EXTRA_OPTIONS ${PYQT4_SIP_FLAGS} -o -a ${CMAKE_BINARY_DIR}/python/qgis.core.api)
ADD_SIP_PYTHON_MODULE(qgis.core core/core.sip qgis_core)

# additional gui includes
INCLUDE_DIRECTORIES(
  ../src/gui
  ../src/gui/symbology-ng
  ../src/plugins
  ${CMAKE_BINARY_DIR}/src/gui
  ${CMAKE_BINARY_DIR}/src/ui
)

# gui module
FILE(GLOB_RECURSE sip_files_gui gui/*.sip)
SET(SIP_EXTRA_FILES_DEPEND ${sip_files_core} ${sip_files_gui})
SET(SIP_EXTRA_OPTIONS ${PYQT4_SIP_FLAGS} -o -a ${CMAKE_BINARY_DIR}/python/qgis.gui.api)

IF(UNIX AND NOT SIP_VERSION_NUM LESS 265984)
  SET(SIP_EXTRA_OPTIONS -P ${SIP_EXTRA_OPTIONS})
  ADD_DEFINITIONS(-Dprotected=public)
ENDIF(UNIX AND NOT SIP_VERSION_NUM LESS 265984)

ADD_SIP_PYTHON_MODULE(qgis.gui gui/gui.sip qgis_core qgis_gui)

# additional analysis includes
INCLUDE_DIRECTORIES(
  ../src/analysis/vector
  ../src/analysis/raster
  ../src/analysis/network
  ../src/analysis/interpolation
  ${CMAKE_BINARY_DIR}/src/analysis/vector
  ${CMAKE_BINARY_DIR}/src/analysis/network
  ${CMAKE_BINARY_DIR}/src/analysis/raster
  ${CMAKE_BINARY_DIR}/src/analysis/interpolation
)

# analysis module
FILE(GLOB sip_files_analysis
  analysis/*.sip
  analysis/raster/*.sip
  analysis/vector/*.sip
  analysis/interpolation/*.sip
)
SET(SIP_EXTRA_FILES_DEPEND ${sip_files_core} ${sip_files_analysis})
SET(SIP_EXTRA_OPTIONS ${PYQT4_SIP_FLAGS} -o -a ${CMAKE_BINARY_DIR}/python/qgis.analysis.api)
ADD_SIP_PYTHON_MODULE(qgis.analysis analysis/analysis.sip qgis_core qgis_analysis)

# network-analysis module
FILE(GLOB_RECURSE sip_files_network_analysis analysis/network/*.sip)
SET(SIP_EXTRA_FILES_DEPEND ${sip_files_core} ${sip_files_network_analysis})
SET(SIP_EXTRA_OPTIONS ${PYQT4_SIP_FLAGS} -o -a ${CMAKE_BINARY_DIR}/python/qgis.networkanalysis.api)
ADD_SIP_PYTHON_MODULE(qgis.networkanalysis analysis/network/networkanalysis.sip qgis_core qgis_networkanalysis)

SET(QGIS_PYTHON_DIR ${PYTHON_SITE_PACKAGES_DIR}/qgis)

IF(WITH_QSCIAPI)
  # wait until after python module builds for api files to be available
  SET(QGIS_PYTHON_API_FILE "${CMAKE_BINARY_DIR}/python/qsci_apis/PyQGIS.api")

  ADD_CUSTOM_TARGET(qsci-api ALL
    DEPENDS python_module_qgis_gui python_module_qgis_core python_module_qgis_analysis python_module_qgis_networkanalysis)

  # run update/concatenate command
  ADD_CUSTOM_COMMAND(TARGET qsci-api
    POST_BUILD
    COMMAND "${CMAKE_COMMAND}" -P "${CMAKE_SOURCE_DIR}/cmake/QsciAPI.cmake"
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
    COMMENT "Generating pyqgis api file" VERBATIM)

  INSTALL(FILES ${QGIS_PYTHON_API_FILE} DESTINATION "${QGIS_DATA_DIR}/python/qsci_apis")

  # create target for generating console auto-completion *.pap binary file
  # takes too long to build (> 1 minute) for targets to have ALL property
  SET(APIS_SRC_DIR "${CMAKE_SOURCE_DIR}/python/qsci_apis")
  SET(APIS_BIN_DIR "${CMAKE_BINARY_DIR}/python/qsci_apis")

  # generate a .pap file to be immediately installed in QGIS source tree (the default .pap)
  ADD_CUSTOM_TARGET(qsci-pap-src
    DEPENDS qsci-api ${QGIS_PYTHON_API_FILE})

  SET(PAP_NAME "pyqgis.pap")
  ADD_CUSTOM_COMMAND(TARGET qsci-pap-src
    POST_BUILD
    COMMAND ${PYTHON_EXECUTABLE} "${APIS_SRC_DIR}/generate_console_pap.py" "${APIS_SRC_DIR}/${PAP_NAME}" "${APIS_SRC_DIR}" "${APIS_BIN_DIR}"
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
    COMMENT "Generating ${PAP_NAME} for console auto-completion (MAY TAKE > 1 MINUTE!)" VERBATIM)
ENDIF(WITH_QSCIAPI)

# Plugin utilities files to copy to staging or install
SET(PY_FILES
  __init__.py
  utils.py
)

ADD_CUSTOM_TARGET(pyutils ALL)
INSTALL(FILES ${PY_FILES} DESTINATION "${QGIS_PYTHON_DIR}")

# stage to output to make available when QGIS is run from build directory
FOREACH(pyfile ${PY_FILES})
  ADD_CUSTOM_COMMAND(TARGET pyutils
    POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy ${pyfile} "${QGIS_PYTHON_OUTPUT_DIRECTORY}"
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${pyfile}
  )
ENDFOREACH(pyfile)

# Byte-compile staged PyQGIS utilities
IF(WITH_PY_COMPILE)
  ADD_CUSTOM_TARGET(pycompile-pyutils ALL
    COMMAND ${PYTHON_EXECUTABLE} -m compileall -q "${PYTHON_OUTPUT_DIRECTORY}/qgis"
    COMMAND ${PYTHON_EXECUTABLE} -m compileall -q "${PYTHON_OUTPUT_DIRECTORY}/console"
    COMMAND ${PYTHON_EXECUTABLE} -m compileall -q "${PYTHON_OUTPUT_DIRECTORY}/pyplugin_installer"
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
    COMMENT "Byte-compiling staged PyQGIS utility modules..."
    DEPENDS pyutils
  )
ENDIF(WITH_PY_COMPILE)
