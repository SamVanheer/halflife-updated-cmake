#### Minimum required version of CMake #########################################

  cmake_minimum_required(VERSION 3.16)


#### Project description #######################################################

  project(HalfLifeMod VERSION 1.0 DESCRIPTION "Half-Life CMake Mod Template" LANGUAGES CXX)


#### CMake custom paths ########################################################

  get_filename_component(STEAM_HL_DIRECTORY "[HKEY_CURRENT_USER\\Software\\Valve\\Steam;ModInstallPath]" ABSOLUTE)

  set(HL_DIRECTORY "${STEAM_HL_DIRECTORY}" CACHE PATH "Path to the Half-Life directory.")
  set(HLDS_DIRECTORY "" CACHE PATH "Path to the Half-Life Dedicated Server directory. Must be specified to automatically set up dedicated server debugging settings.")

  if ("${HL_DIRECTORY}" STREQUAL "")
    message(FATAL_ERROR "You must specify the path to the Half-Life directory.")
  elseif (NOT EXISTS ${HL_DIRECTORY})
    message(FATAL_ERROR "The given Half-Life directory path ${HL_DIRECTORY} is not valid.")
  endif()


#### Shared libraries ##########################################################

  #### Add "vgui" library

    add_library(vgui SHARED IMPORTED)

    if (WIN32)
      set_target_properties(vgui PROPERTIES IMPORTED_IMPLIB "${CMAKE_CURRENT_SOURCE_DIR}/utils/vgui/lib/win32_vc6/vgui.lib")
    else()
      set_target_properties(vgui PROPERTIES IMPORTED_LOCATION "${HL_DIRECTORY}/vgui.so")
    endif()

    target_include_directories(vgui INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/utils/vgui/include)


  #### Add "SDL2" library

    add_library(SDL2 SHARED IMPORTED)

    if (WIN32)
      set_target_properties(SDL2 PROPERTIES IMPORTED_IMPLIB "${CMAKE_CURRENT_SOURCE_DIR}/lib/public/SDL2.lib")
    else()
      set_target_properties(SDL2 PROPERTIES IMPORTED_LOCATION "${HL_DIRECTORY}/libSDL2.so")
    endif()


  #### Add "client" (cl_dll/) library

    add_library(client SHARED)

    set_target_properties(client PROPERTIES PREFIX "")
    set_target_properties(client PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

    target_include_directories(client PRIVATE dlls cl_dll public common pm_shared engine utils/vgui/include game_shared external)
    target_compile_definitions(client PRIVATE _CRT_SECURE_NO_WARNINGS $<$<CONFIG:DEBUG>:_DEBUG> CLIENT_DLL CLIENT_WEAPONS HL_DLL)
    target_compile_options(client PRIVATE $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:AppleClang>,$<CXX_COMPILER_ID:GNU>>: -fpermissive -fno-strict-aliasing> $<$<CXX_COMPILER_ID:MSVC>: /W3 /Zc:threadSafeInit->)
    target_link_libraries(client PRIVATE vgui $<$<PLATFORM_ID:Windows>:wsock32> SDL2)

    add_subdirectory(cl_dll)


  #### Add "server" (dlls/) library

    add_library(server SHARED dlls/hl.def)

    set_target_properties(server PROPERTIES PREFIX "")
    set_target_properties(server PROPERTIES MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

    target_include_directories(server PRIVATE dlls engine common pm_shared game_shared public)
    target_compile_definitions(server PRIVATE _CRT_SECURE_NO_WARNINGS $<$<CONFIG:DEBUG>:_DEBUG> QUIVER VOXEL QUAKE2 VALVE_DLL CLIENT_WEAPONS)
    target_compile_options(server PRIVATE $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:AppleClang>,$<CXX_COMPILER_ID:GNU>>: -fpermissive -fno-strict-aliasing> $<$<CXX_COMPILER_ID:MSVC>: /W3 /Zc:threadSafeInit->)

    add_subdirectory(dlls)


#### Visual Studio #############################################################

  #### Create filters

    get_target_property(SERVER_SOURCES server SOURCES)
    get_target_property(CLIENT_SOURCES client SOURCES)
    source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR} FILES ${SERVER_SOURCES})
    source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR} FILES ${CLIENT_SOURCES})


  #### Set starting project

    set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT client)


  #### Set debugging settings to work on the mod directory

    if (WIN32)
      set(HL_EXECUTABLE "hl.exe")
      set(HLDS_EXECUTABLE "hlds.exe")
    else()
      set(HL_EXECUTABLE "hl_linux")
      set(HLDS_EXECUTABLE "hlds_linux")
    endif()

    get_filename_component(MOD_NAME ${CMAKE_INSTALL_PREFIX} NAME_WE)

    set_target_properties(client PROPERTIES
      VS_DEBUGGER_COMMAND "${HL_DIRECTORY}/${HL_EXECUTABLE}"
      VS_DEBUGGER_COMMAND_ARGUMENTS "-game ${MOD_NAME}"
      VS_DEBUGGER_WORKING_DIRECTORY "${HL_DIRECTORY}"
    )

    set_target_properties(server PROPERTIES
      VS_DEBUGGER_COMMAND "${HLDS_DIRECTORY}/${HLDS_EXECUTABLE}"
      VS_DEBUGGER_COMMAND_ARGUMENTS "-game ${MOD_NAME}"
      VS_DEBUGGER_WORKING_DIRECTORY "${HLDS_DIRECTORY}"
    )


  #### Set install paths to the user-defined mod directory

    install(TARGETS client RUNTIME DESTINATION cl_dlls LIBRARY DESTINATION cl_dlls)
    install(FILES $<TARGET_PDB_FILE:client> DESTINATION cl_dlls OPTIONAL)

    install(TARGETS server RUNTIME DESTINATION dlls LIBRARY DESTINATION dlls)
    install(FILES $<TARGET_PDB_FILE:server> DESTINATION dlls OPTIONAL)
