
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/CMakeModules")

if (WIN32)

cmake_minimum_required(VERSION 3.11)
project("Taiwu_Mods" CSharp)
set(CMAKE_CSharp_FLAGS "/langversion:6")
include(CSharpUtilities)

SET(DLLPATH "The Scroll Of Taiwu Alpha V1.0_Data/Managed/")
endif()

if(UNIX)
cmake_minimum_required(VERSION 2.6.2)
project(Taiwu_Mods NONE)

# Define generator properties
set(VS10SLN OFF CACHE BOOL "Force generation of SLN.")

# Find C#
find_package( CSharp REQUIRED )
include( ${CSHARP_USE_FILE} )

# Find MSBuild
find_package( MSBuild REQUIRED )
include( ${MSBuild_USE_FILE} )

# Setup build locations.
if(NOT CMAKE_RUNTIME_OUTPUT_DIR)
  set(CMAKE_RUNTIME_OUTPUT_DIR ${CMAKE_CURRENT_BINARY_DIR}/bin)
endif()
if(NOT CMAKE_LIBRARY_OUTPUT_DIR)
  set(CMAKE_LIBRARY_OUTPUT_DIR ${CMAKE_CURRENT_BINARY_DIR}/lib)
endif()

# Make sure the nested directory structure exists
file(MAKE_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIR})
file(MAKE_DIRECTORY ${CMAKE_LIBRARY_OUTPUT_DIR})

# If no build type is set it should defaults to debug
if( NOT CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Debug CACHE STRING "Choose a build type: [Debug Release]" FORCE )
endif()

# Set some default build flags.
set(CSBUILDFLAGS "")
set(MSBUILDFLAGS "")

# Set up the compiler flag for Debug/Release mode.
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
  list(APPEND CSBUILDFLAGS "/debug:full" "/debug+" "/optimize-")
  list(APPEND MSBUILDFLAGS "/p:Configuration=Debug")
else()
  list(APPEND CSBUILDFLAGS "/debug-" "/optimize+")
  list(APPEND MSBUILDFLAGS "/p:Configuration=Release")
endif()

# Define whether to use assembly deleyed signing or not
set(DELAY_SIGN OFF CACHE BOOL "Enable delayed assembly signing.")
if(DELAY_SIGN)
  # The if-stmt converts On Y, YES to bool and from here on we want only a real
  # true or false.
  set(DELAY_SIGN "True") 
  list(APPEND CSBUILDFLAGS "/delaysign+" "/keyfile:${CMAKE_SOURCE_DIR}/public.snk")
else()
  set(DELAY_SIGN "False")
endif(DELAY_SIGN)

# Suppress the warnings coming from the /doc switch. There is no point warning 
# about missing documentation for certain methods. We all know about it :)
# 1591 - Missing documentation
# 1587 - XML comment not placed on a valid language element.
list(APPEND CSBUILDFLAGS "/nowarn:1591,1587")


# The lock is used to avoid regeneration of the sln-s and csproj-s if the
# build system is steered by another one. For example if TaiwuModsOpt is embedded
# as a vendor to say SolidReflector. 
get_property(SLN_GEN_LOCK GLOBAL PROPERTY SLN_GEN_LOCK_PROPERTY)
if(NOT SLN_GEN_LOCK)
  set_property(GLOBAL PROPERTY SLN_GEN_LOCK_PROPERTY "TaiwuModsOpt")
endif()

	SET(DLLPATH "${CMAKE_CURRENT_LIST_DIR}/../dlls/")
	SET(STEAMDIR "")
endif(UNIX)


include(CMakeModules/JSONParser.cmake)


#遍历Mod 目录
MACRO(SUBDIRLIST result curdir)
  #message(STATUS "search sub dir in:" ${curdir})
  FILE(GLOB children RELATIVE ${curdir} ${curdir}/*)
  SET(dirlist "")
  FOREACH(child ${children})
    IF(IS_DIRECTORY ${curdir}/${child} AND  (NOT ${child} STREQUAL "build") AND (NOT ${child} STREQUAL ".git") AND (NOT ${child} STREQUAL "CMakeModules") AND (NOT ${child} STREQUAL ".travis") AND (EXISTS ${curdir}/${child}/Info.json))
      message(STATUS "Found Mod: " ${child})
      LIST(APPEND dirlist ${child})
    ENDIF()
  ENDFOREACH()
  SET(${result} ${dirlist})
ENDMACRO()

if(UNIX)
#为每个目录生成Mod
MACRO(CREATEMOD moddir)
	SET(FILELIST "")
	#遍历所有.cs文件
	FILE(GLOB_RECURSE FILES ${moddir}/*.cs)
	FOREACH(csfile ${FILES})
		  LIST(APPEND FILELIST ${csfile})
	ENDFOREACH()

	#message(STATUS ${moddir} " file list:" ${FILELIST})

	set(dlldepends "")
	
	#遍历mod目录下dll，加入引用
	FILE(GLOB_RECURSE MODDLLFILES ${moddir}/*.dll)
	FOREACH(dllfile ${MODDLLFILES})
	  get_filename_component(dllfname ${dllfile} NAME)
	  string(REGEX REPLACE "\\.[^.]*$" "" dllname ${dllfname})
	  message(STATUS "Add dll type ${dllname}  from ${dllfile}   for mod ${moddir}")
	  LIST(APPEND dlldepends ${dllfile})
	ENDFOREACH()
	
	# 解析mod目录下 Info.json 文件， 生成对应 Properties/AssemblyInfo.cs ,添加mod依赖
	file(STRINGS ${moddir}/Info.json infojson)
	set(modinfo "")
	set(DisplayName "")
	set(Author "")
	set(Version "")
	set(Requirements "")
	set(modinfo.Requirements "")
	sbeParseJson(modinfo infojson)

	# Now you can use parsed variables.
	#foreach(var ${modinfo})
	#	message("vals : ${var} = ${${var}}")
	#endforeach()

	set(DisplayName ${modinfo.DisplayName})
	set(Author ${modinfo.Author})
	set(Version ${modinfo.Version})
	set(Requirements ${modinfo.Requirements})
	foreach(var ${Requirements})
		message(STATUS "requirements : ${modinfo.Requirements_${var}}")
		LIST(APPEND dlldepends  ${CMAKE_LIBRARY_OUTPUT_DIR}/${modinfo.Requirements_${var}}.dll)
	endforeach()
	
	#创建项目
	csharp_add_library(
		${moddir}.dll
		A.cs
		${FILELIST}
	)
		
	csharp_add_project_meta(
		${moddir}.dll
		"TargetFrameworkVersion" "2"
    )

	#项目基础引用
	csharp_add_dependencies(
		${moddir}.dll
		System
		System.Core
		System.Data
		System.Xml
		System.Runtime.Serialization
		${STEAMDIR}${DLLPATH}/Assembly-CSharp.dll
		${STEAMDIR}${DLLPATH}0Harmony12.dll
		${STEAMDIR}${DLLPATH}Newtonsoft.Json.dll
		${STEAMDIR}${DLLPATH}UnityEngine.dll
		${STEAMDIR}${DLLPATH}UnityEngine.CoreModule.dll
		${STEAMDIR}${DLLPATH}UnityEngine.ImageConversionModule.dll
		${STEAMDIR}${DLLPATH}UnityEngine.TextRenderingModule.dll
		${STEAMDIR}${DLLPATH}UnityEngine.IMGUIModule.dll
		${STEAMDIR}${DLLPATH}UnityEngine.UI.dll
		${STEAMDIR}${DLLPATH}UnityEngine.UIElementsModule.dll
		${STEAMDIR}${DLLPATH}UnityEngine.UIModule.dll
		${dlldepends}
    )

	set_property(GLOBAL APPEND PROPERTY target_mod_version_property "${modinfo.Version}")

	# When you are done, clean parsed variables
	sbeClearJson(example)
	
ENDMACRO()
ENDIF(UNIX)

#win+vs
IF(WIN32)

#为每个目录生成Mod
MACRO(CREATEMOD moddir)

	# 解析mod目录下 Info.json 文件， 生成对应 Properties/AssemblyInfo.cs ,添加mod依赖
	#读取文件
	file(STRINGS ${moddir}/Info.json infojson)
	#初始化变量
	set(modinfo "")
	set(Id "")
	set(DisplayName "")
	set(Author "")
	set(Version "")
	set(Requirements "")
	set(modinfo.Requirements "")
	#解析json
	sbeParseJson(modinfo infojson)

	# 调试代码：列出json值
	#foreach(var ${modinfo})
	#	message("vals : ${var} = ${${var}}")
	#endforeach()
	#取json值
	set(Id ${modinfo.Id})
	set(DisplayName ${modinfo.DisplayName})
	set(Author ${modinfo.Author})
	set(Version ${modinfo.Version})
	set(Requirements ${modinfo.Requirements})

	message(STATUS "Found Mod: ${Id}  ${DisplayName} Author ${Author}  Version ${Version} ${CMAKE_CURRENT_BINARY_DIR} ")
	
	SET(FILELIST "")
	#遍历所有.cs文件
	FILE(GLOB_RECURSE FILES ${moddir}/*.cs)
	FOREACH(csfile ${FILES})
		  LIST(APPEND FILELIST ${csfile})
	ENDFOREACH()

	set(assemblyinfo "using System\;
using System.Diagnostics\;
using System.Reflection\;
using System.Runtime.CompilerServices\;
using System.Runtime.InteropServices\;
using System.Runtime.Versioning\;

[assembly: AssemblyVersion(\"${Version}\")]
[assembly: CompilationRelaxations(8)]
[assembly: RuntimeCompatibility(WrapNonExceptionThrows = true)]
[assembly: AssemblyTitle(\"${Id}\")]
[assembly: AssemblyDescription(\"${DisplayName}\")]
[assembly: AssemblyCompany(\"TaiwuMods\")]
[assembly: AssemblyProduct(\"${Id}\")]
[assembly: AssemblyCopyright(\"Copyright ©  2018\")]
[assembly: AssemblyFileVersion(\"${Version}\")]
")

     FILE(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${moddir}/Properties/AssemblyInfo.cs" ${assemblyinfo})
	 LIST(APPEND FILELIST "${CMAKE_CURRENT_BINARY_DIR}/${moddir}/Properties/AssemblyInfo.cs")
	#message(STATUS ${moddir} " file list:" ${FILELIST})

	#创建项目
	add_library(${moddir}
		SHARED
		${FILELIST}
		${moddir}/Info.json
		)
	#项目版本 .net 3.5
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_TARGET_FRAMEWORK_VERSION "v3.5")
	#项目基础引用
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCES
		"System"
		"System.Core"
		"System.Data"
		"System.Xml"
		"System.Runtime.Serialization"
		)
	#AssemblyInfo.cs 处理
	csharp_set_designer_cs_properties(
		"${CMAKE_CURRENT_BINARY_DIR}/${moddir}/Properties/AssemblyInfo.cs"
	)
	
	#dll 引用
	#游戏主dll
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_Assembly-CSharp "${CMAKE_CURRENT_LIST_DIR}\\Assembly-CSharp.dll")
	# harmonypatch工具
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_Harmony12 "${STEAMDIR}${DLLPATH}0harmony12.dll")
	#其他游戏相关dll
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_Newtonsoft.Json "${STEAMDIR}${DLLPATH}Newtonsoft.Json.dll")
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_UnityEngine "${STEAMDIR}${DLLPATH}UnityEngine.dll")
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_UnityEngine.CoreModule "${STEAMDIR}${DLLPATH}UnityEngine.CoreModule.dll")
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_UnityEngine.ImageConversionModule "${STEAMDIR}${DLLPATH}UnityEngine.ImageConversionModule.dll")
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_UnityEngine.TextRenderingModule "${STEAMDIR}${DLLPATH}UnityEngine.TextRenderingModule.dll")
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_UnityEngine.IMGUIModule "${STEAMDIR}${DLLPATH}UnityEngine.IMGUIModule.dll")
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_UnityEngine.UI "${STEAMDIR}${DLLPATH}UnityEngine.UI.dll")
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_UnityEngine.UIElementsModule "${STEAMDIR}${DLLPATH}UnityEngine.UIElementsModule.dll")
	set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_UnityEngine.UIModule "${STEAMDIR}${DLLPATH}UnityEngine.UIModule.dll")
	
	#遍历mod目录下dll，加入引用
	FILE(GLOB_RECURSE MODDLLFILES ${moddir}/*.dll)
	FOREACH(dllfile ${MODDLLFILES})
	  get_filename_component(dllfname ${dllfile} NAME)
	  string(REGEX REPLACE "\\.[^.]*$" "" dllname ${dllfname})
	  message(STATUS "	Add dll type ${dllname}  from ${dllfile}   for mod ${moddir}")
	  set_property(TARGET ${moddir} PROPERTY VS_DOTNET_REFERENCE_${dllname} "${dllfile}")
	ENDFOREACH()
	


	#设置项目引用
	set(ProjectDeps "")
	set(var "")
	foreach(var ${Requirements})
		message(STATUS "	Mod requirements : ${modinfo.Requirements_${var}}")
		target_link_libraries(${moddir} ${modinfo.Requirements_${var}})
	endforeach()

	#清除json
	sbeClearJson(example)
	
	#生成后事件，copy dll到游戏Mods目录
	ADD_CUSTOM_COMMAND(TARGET ${moddir}
		POST_BUILD
		COMMAND if not exist "${STEAMDIR}Mods\\${moddir}/" echo not copy file & goto END
		COMMAND copy "$(TargetPATH)" "${STEAMDIR}Mods\\${moddir}/"
		COMMAND powershell -NoProfile -ExecutionPolicy Unrestricted -File "..\\..\\CopyMod.ps1" -Mod ${moddir} -ModsDir "${STEAMDIR}Mods"
		COMMAND echo "copy dll file ${moddir}.dll ok"
		COMMENT "copy dll file to steam game"
		COMMAND :END
	)

ENDMACRO()
ENDIF(WIN32)

message(STATUS "Finding All subdir that contains mods")

SUBDIRLIST(SUBDIRS ${CMAKE_CURRENT_LIST_DIR})
#message(STATUS "subdir list:" ${SUBDIRS})
FOREACH(subdir ${SUBDIRS})
  CREATEMOD(${subdir})
ENDFOREACH()

IF(UNIX)
# Resolve all target dependencies
csharp_resolve_dependencies()

get_property(SLN_GEN_LOCK GLOBAL PROPERTY SLN_GEN_LOCK_PROPERTY)
if (SLN_GEN_LOCK STREQUAL "TaiwuModsOpt")
  # Save solution
  csharp_save_vs_solution(TaiwuModsOpt)
endif()

ENDIF(UNIX)
