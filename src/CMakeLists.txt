include_directories(${CMAKE_CURRENT_SOURCE_DIR})

################################################################################
# Define some commons compile flags.                                           #
################################################################################

set(CMAKE_CXX_STANDARD 14)
# MSVC specific compile definitions
set(MSVC_COMPILE_DEFINITIONS _USE_MATH_DEFINES NOMINMAX)

################################################################################
# Init the list of libraries that JSBSim links with                            #
################################################################################

# MSVC and MINGW linked libraries
set(WINDOWS_LINK_LIBRARIES wsock32 ws2_32)
# Unix linked libraries
set(UNIX_LINK_LIBRARIES m)


################################################################################
# Build and install libraries                                                  #
################################################################################

set(VERSION_MESSAGE)
if(DEFINED ENV{TRAVIS})
  set(VERSION_MESSAGE " [Travis build $ENV{TRAVIS_BUILD_NUMBER}/commit $ENV{TRAVIS_COMMIT}]")
elseif(DEFINED ENV{APPVEYOR})
  set(VERSION_MESSAGE " [AppVeyor build $ENV{APPVEYOR_BUILD_NUMBER}/commit $ENV{APPVEYOR_REPO_COMMIT}]")
elseif(DEFINED ENV{GITHUB_RUN_NUMBER})
  set(VERSION_MESSAGE " [GitHub build $ENV{GITHUB_RUN_NUMBER}/commit $ENV{GITHUB_SHA}]")
endif()

# Compile definitions common to all platforms
set(COMPILE_DEFINITIONS JSBSIM_VERSION="${PROJECT_VERSION}${VERSION_MESSAGE}")

set(STATIC_LIBNAME_SUFFIX)
if(MSVC AND BUILD_SHARED_LIBS)
  set(STATIC_LIBNAME_SUFFIX "static")
endif()

add_subdirectory(initialization)
add_subdirectory(input_output)
add_subdirectory(math)
add_subdirectory(models)
add_subdirectory(simgear)
add_subdirectory(GeographicLib)

set(HEADERS FGFDMExec.h
            FGJSBBase.h)
set(SOURCES FGFDMExec.cpp
            FGJSBBase.cpp)

add_library(libJSBSim ${HEADERS} ${SOURCES}
  $<TARGET_OBJECTS:Init>
  $<TARGET_OBJECTS:Atmosphere>
  $<TARGET_OBJECTS:FlightControl>
  $<TARGET_OBJECTS:Propulsion>
  $<TARGET_OBJECTS:Models>
  $<TARGET_OBJECTS:Math>
  $<TARGET_OBJECTS:InputOutput>
  $<TARGET_OBJECTS:Properties>
  $<TARGET_OBJECTS:Xml>
  $<TARGET_OBJECTS:Magvar>
  $<TARGET_OBJECTS:Misc>
  $<TARGET_OBJECTS:IOStreams>
  $<TARGET_OBJECTS:Structure>
  $<TARGET_OBJECTS:Simgear>
  $<TARGET_OBJECTS:GeographicLib>
  )

if(EXPAT_FOUND)
  include_directories(${EXPAT_INCLUDE_DIRS})
  if (PKG_CONFIG_FOUND)
    target_link_libraries(libJSBSim ${PC_EXPAT_LIBRARIES})
    set(ALL_LINK_LIBRARIES ${PC_EXPAT_LIBRARIES})
  else()
    target_link_libraries(libJSBSim ${EXPAT_LIBRARIES})
    set(ALL_LINK_LIBRARIES ${EXPAT_LIBRARIES})
  endif()
else()
  list(APPEND COMPILE_DEFINITIONS HAVE_EXPAT_CONFIG_H)
endif()

# Manage compile definitions
set(JSBSIM_COMPILE_DEFINITIONS ${COMPILE_DEFINITIONS} PARENT_SCOPE)
set(JSBSIM_MSVC_COMPILE_DEFINITIONS ${MSVC_COMPILE_DEFINITIONS} PARENT_SCOPE)

if(MSVC)
  foreach(COMPILE_DEF ${MSVC_COMPILE_DEFINITIONS})
    target_compile_definitions(libJSBSim PUBLIC ${COMPILE_DEF})
    target_compile_definitions(Misc PUBLIC ${COMPILE_DEF})
  endforeach(COMPILE_DEF)
endif(MSVC)

foreach(COMPILE_DEF ${COMPILE_DEFINITIONS})
  target_compile_definitions(libJSBSim PUBLIC ${COMPILE_DEF})
  target_compile_definitions(Xml PUBLIC ${COMPILE_DEF})
endforeach(COMPILE_DEF)

set_target_properties (libJSBSim PROPERTIES
                                 OUTPUT_NAME JSBSim${STATIC_LIBNAME_SUFFIX}
                                 VERSION ${LIBRARY_VERSION}
                                 TARGET_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})

# Manage link libraries
set(JSBSIM_LINK_LIBRARIES ${ALL_LINK_LIBRARIES} PARENT_SCOPE)
set(JSBSIM_WINDOWS_LINK_LIBRARIES ${WINDOWS_LINK_LIBRARIES} PARENT_SCOPE)
set(JSBSIM_UNIX_LINK_LIBRARIES ${UNIX_LINK_LIBRARIES} PARENT_SCOPE)

if(WIN32 AND (MSVC OR MINGW))
  foreach(LINK_LIBRARY ${WINDOWS_LINK_LIBRARIES})
    target_link_libraries(libJSBSim ${LINK_LIBRARY})
  endforeach()
elseif(UNIX)
  foreach(LINK_LIBRARY ${UNIX_LINK_LIBRARIES})
    target_link_libraries(libJSBSim ${LINK_LIBRARY})
  endforeach()
endif()

if(BUILD_SHARED_LIBS)
  set_target_properties (libJSBSim PROPERTIES
                                   SOVERSION ${LIBRARY_SOVERSION}
                                   FRAMEWORK ON)
  install(TARGETS libJSBSim LIBRARY DESTINATION lib
                            NAMELINK_SKIP
                            COMPONENT runtime
                            # For Mac
                            FRAMEWORK DESTINATION "/Library/Frameworks")
  install(TARGETS libJSBSim LIBRARY DESTINATION lib
                            NAMELINK_ONLY
                            COMPONENT devel
                            # For Mac
                            FRAMEWORK DESTINATION "/Library/Frameworks")
else()
  install(TARGETS libJSBSim ARCHIVE DESTINATION lib
                            COMPONENT devel
                            # For Mac
                            FRAMEWORK DESTINATION "/Library/Frameworks")
endif()

################################################################################
# Build and install command line executable                                    #
################################################################################

add_executable(JSBSim JSBSim.cpp)
target_link_libraries(JSBSim libJSBSim)

install(TARGETS JSBSim RUNTIME DESTINATION bin COMPONENT runtime)
install(FILES ${HEADERS} DESTINATION include/JSBSim COMPONENT devel)
