#
# Inputs:
# HALIDE_DIR
# HOST_CLANG
# HALIDE_DEBUG
# HALIDE_PROFILE
#
# Functions.txt


cmake_minimum_required(VERSION 3.1)

if (NOT HALIDE_DIR)
    if ($ENV{HALIDE_DIR})
        set(HALIDE_DIR, $ENV{HALIDE_DIR})
    else ()
        find_file(HALIDE_HEADER NAMES Halide.h PATHS ../../halide ../halide ../../../halide ../../../Halide PATH_SUFFIXES include)
        if (HALIDE_HEADER)
            get_filename_component(HALIDE_DIR ${HALIDE_HEADER} DIRECTORY)
            get_filename_component(HALIDE_DIR ${HALIDE_DIR} DIRECTORY)
        else ()
            message(FATAL_ERROR "HALIDE_DIR not set")
        endif ()
    endif ()
endif ()

if (NOT EXISTS "${HALIDE_DIR}/include/Halide.h")
    message(FATAL_ERROR "HALIDE_DIR ${HALIDE_DIR} does not contain include/Halide.h")
endif ()

set(HALIDE_DIR ${HALIDE_DIR} CACHE PATH "path to halide")
message(STATUS "HALIDE_DIR is ${HALIDE_DIR}")

if (HOST_CLANG)
    set(clangCompiler "${HOST_CLANG}")
elseif (ENV{HOST_CLANG})
    set(clangCompiler "$ENV{HOST_CLANG}")
elseif (CLANG)
    set(clangCompiler "${CLANG}")
elseif (ENV{CLANG})
    set(clangCompiler "$ENV{CLANG}")
else ()
    find_program(clangCompiler "clang++")
endif ()

if (NOT clangCompiler)
    message(FATAL_ERROR "HOST_CLANG not set")
endif ()


find_file(hlLibDll NAMES Halide.dll libHalide.so PATHS ${HALIDE_DIR}/bin/Release ${HALIDE_DIR}/Release ${HALIDE_DIR}/bin ${HALIDE_DIR}/lib ${HALIDE_DIR}/lib/Release ../../../Halide/bin)
find_file(hlLib NAMES Halide.lib libHalide.so PATHS    ${HALIDE_DIR}/lib/Release ${HALIDE_DIR}/Release ${HALIDE_DIR}/bin ${HALIDE_DIR}/lib ${HALIDE_DIR}/lib/Release ../../../Halide/lib)

if(NOT hlLibDll)
    message(STATUS "libHalide.so, Halide.dll not found in ${HALIDE_DIR} , trying other locations")
    set(hlLibDll ../../../Halide/lib/libHalide.a)
endif()

#message(FATAL_ERROR "not-found ${hlLib} ")
set(SHELL_CMD )
if(CMAKE_HOST_WIN32 AND ${hlLibDll} MATCHES ".+\\.so$")
	set(WINDOWS_LINUX_SUBSYS ON)
    message(STATUS "Using Linux subsystem (found ${hlLibDll})")
    set(SHELL_CMD $ENV{WINDIR}/System32/bash.exe)
	#find_program(SHELL_CMD bash) # conflicts with Git/usr/bin/bash.exe
	message(STATUS "Using bash ${SHELL_CMD}")
endif()
	
if (NOT CMAKE_HOST_WIN32 OR WINDOWS_LINUX_SUBSYS)
    set(clangCompilerFlags "-g -std=c++11 -lpthread -ltinfo -ldl -lz -fno-rtti")
    set(hlLib -lHalide)
else ()
    set(clangCompilerFlags "-g -std=c++14 -Wno-inconsistent-dllimport -Wl,/ignore:4217,/ignore:4049,/STACK:999999999")
endif ()


if (WIN32 AND NOT MINGW)
    set(DOTLIB ".lib")
else ()
    set(DOTLIB ".a")
endif ()

if (CMAKE_HOST_WIN32)
    set(HOSTDOTEXE ".exe")
else ()
    set(HOSTDOTEXE "")
endif ()

set(hlFeatures )
# map CMAKE_SYSTEM_PROCESSOR to HL_TARGET `arch-bits`
if (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "aarch64")
    set(hlArchBits arm-64)
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "armv7-a")
    set(hlArchBits arm-32)
    set(hlFeatures "${hlFeatures}-armv7s") # arm superset with FP
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "armabi" OR ${CMAKE_SYSTEM_PROCESSOR} STREQUAL "armv5te")
    set(hlArchBits "arm-32")
elseif (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "x86_64" OR ${CMAKE_SYSTEM_PROCESSOR} STREQUAL "AMD64")
    set(hlArchBits "x86-64")
    set(hlFeatures ${hlFeatures}-sse41)
else ()
    message(FATAL_ERROR "unsupported arch '${CMAKE_SYSTEM_PROCESSOR}'")
endif ()

if (ANDROID)
    set(hlTarget ${hlArchBits}-android)
elseif(WINDOWS_LINUX_SUBSYS)
	set(hlTarget ${hlArchBits}-windows)
else()
    set(hlTarget ${hlArchBits}) # using host OS
endif ()

if (MINGW)
    set(hlTarget ${hlTarget}-mingw)
endif()

set(hlTarget "${hlTarget}${hlFeatures}")

#message(FATAL_ERROR ${CMAKE_BUILD_TYPE})
#if (CMAKE_BUILD_TYPE EQUAL "DEBUG")
#	set(HALIDE_DEBUG ON)
#endif()

if (HALIDE_DEBUG)
    set(hlTarget ${hlTarget}-debug)
endif ()

# currently no profiling on windows, this will includes parts of the runtime and cant get the linker to accept weak linkage TODO
if (HALIDE_PROFILE)
    set(hlTarget ${hlTarget}-profile)
endif ()

#set(HALIDE_AUTO_SCHEDULE ON)

if (HALIDE_AUTO_SCHEDULE)
    set(hlAutoSchedule auto_schedule=true)
    set(hlExtraEmit ,schedule)
    set(hlTargetDirSuffix ${hlTargetDirSuffix}_auto_schedule)
    # use macro AUTO_SCHEDULE to disable scheduling related calls in code outside generators
    set(clangCompilerFlags "${clangCompilerFlags} -DAUTO_SCHEDULE")
endif ()
set(hlExtraEmit ,schedule)

# only include the runtime once (fix for windows)
set(hlTarget ${hlTarget}-no_runtime)

set(HLPIPES_ROOT ${CMAKE_SOURCE_DIR}/${HLPIPES_ROOT})

# source-tree paths
get_filename_component(hlGeneratorSourcesDir ${HLPIPES_ROOT} ABSOLUTE)
set(hlHeaderExportDir ${hlGeneratorSourcesDir}/h)


# build paths
get_filename_component(hlBinDir ${CMAKE_CURRENT_BINARY_DIR} ABSOLUTE)
set(hlGenDir ${hlBinDir}/gen)
set(hlGenToolCppRel gen/tool.cpp)
set(hlGenToolCpp ${hlGenDir}/tool.cpp)
set(hlTargetDir ${hlBinDir}/${hlTarget}${hlTargetDirSuffix}) # hlTarget must be a child of hlBinDir!


# generate/copy/download required files
file(COPY ${hlLibDll} DESTINATION ${hlBinDir})
file(MAKE_DIRECTORY ${hlGenDir})
# TODO add sha256 file check
file(DOWNLOAD https://gist.githubusercontent.com/f4bsch/cbf5710c97974e97b2147619f7be2f08/raw/75be2518e2a374485f3ae08602be20034d730fb9/HalideGenGen ${hlGenToolCpp})
file(WRITE ${hlGenDir}/dummy.c "int hl_pipelines_dummy_noop() {return 0;}")
file(MAKE_DIRECTORY "${hlTargetDir}")
file(MAKE_DIRECTORY "${hlHeaderExportDir}")


# make sure headers are generated before building:
add_custom_target(hl_pipelines_headers ALL)

set(hlPipelineLibs "")
set(hlPipelineHeaders "")


file(GLOB hlGeneratorSources ${HLPIPES_ROOT}/*_generator.cpp)
file(GLOB hlPipelineMiscSources ${HLPIPES_ROOT}/*.cpp)
file(GLOB hlPipelineMiscHeaders ${HLPIPES_ROOT}/*.h)

function(generator_tag output input)
    get_filename_component(tag ${input} NAME_WE)
    string(LENGTH ${tag} len)
    math(EXPR len "${len} - 10")
    string(SUBSTRING ${tag} 0 ${len} tag)
    SET(${output} ${tag} PARENT_SCOPE)
endfunction()

function(translate_paths output input)
	#string(REPLACE "/" "\\" rp ${rp})
	set(tp ${input})
    string(REPLACE "A:/" "/mnt/a/" tp ${tp})
	string(REPLACE "B:/" "/mnt/b/" tp ${tp})
	string(REPLACE "C:/" "/mnt/c/" tp ${tp})
	string(REPLACE "D:/" "/mnt/d/" tp ${tp})
	SET(${output} ${tp} PARENT_SCOPE)
endfunction()


# Filtering
foreach (source IN LISTS hlGeneratorSources)
    list(REMOVE_ITEM hlPipelineMiscSources ${source})
endforeach ()


set(cmakeCmd ${CMAKE_COMMAND})

string(REPLACE ";" "\t" hlPipelineMiscSourcesJoin "${hlPipelineMiscSources}")
set(hlGenCommonCompilerArgs "${hlPipelineMiscSourcesJoin} ${hlGenToolCppRel} ${hlLib} ${clangCompilerFlags} -I${HALIDE_DIR}/include -L${HALIDE_DIR}/lib")

if (CMAKE_HOST_WIN32 AND NOT WINDOWS_LINUX_SUBSYS)
    set(compileGen "\"${clangCompiler}\" ${hlGeneratorSourcesDir}/%gen%_generator.cpp ${hlGenCommonCompilerArgs} -o %gen%_gen.exe")
    set(cmdGenFuncHeader ${hlTarget}${hlTargetDirSuffix}/gen_func_h.cmd)
    file(WRITE ${hlBinDir}/${cmdGenFuncHeader}
            "@echo off
set gen=%1\n set func=%2\n set genArgs=%3\n set feat=%4\n
if not exist \"${hlHeaderExportDir}/%func%.h\" (
	echo generating %func%.h with args='%genArgs:\"=%' feat='%feat%'
	if not exist %gen%_gen.exe ${compileGen}
	\"%gen%_gen.exe\" -g %gen% -f %func% -o ${hlTarget}${hlTargetDirSuffix} -e h target=${hlTarget}%feat% %genArgs:\"=% ${hlAutoSchedule} || del %gen%_gen.exe
	\"${CMAKE_COMMAND}\" -E copy_if_different \"${hlTarget}${hlTargetDirSuffix}/%func%.h\" \"${hlHeaderExportDir}\"
)
")

    set(cmdGenFuncLib ${hlTarget}${hlTargetDirSuffix}/gen_func.cmd)
    file(WRITE ${hlBinDir}/${cmdGenFuncLib}
            "@echo off
set gen=%1\n set func=%2\n set genArgs=%3\n set feat=%4\n
(
	echo generating %func% for target '${hlTarget}%feat:\"=%' with args '%genArgs:\"=%'
REM	${compileGen}
	REM if exist ${hlTarget}${hlTargetDirSuffix}/%func%.lib del ${hlTarget}${hlTargetDirSuffix}/%func%.lib
	\"%gen%_gen.exe\" -g %gen% -f %func% -o ${hlTarget}${hlTargetDirSuffix} -e h,static_library,cpp,html${hlExtraEmit} %genArgs:\"=% target=${hlTarget}%feat:\"=% ${hlAutoSchedule} || (echo \"generator failed (%errorlevel%)\" 1>&2 && exit 1)
	\"${CMAKE_COMMAND}\" -E copy_if_different \"${hlTarget}${hlTargetDirSuffix}/%func%.h\" \"${hlHeaderExportDir}\"
)
")


    set(cmdGenGen ${hlTarget}${hlTargetDirSuffix}/gen_gen.cmd)
    file(WRITE ${hlBinDir}/${cmdGenGen}
            "@echo off
set gen=%1\n
( ${compileGen} )
")
else()
	translate_paths(compileGen "clang++ ${hlGeneratorSourcesDir}/\${gen}_generator.cpp ${hlGenCommonCompilerArgs} -o \${gen}_gen")
    translate_paths(hlHeaderExportDirT ${hlHeaderExportDir})
	
	set(cmdGenFuncHeader ${hlTarget}${hlTargetDirSuffix}/gen_func_h.sh)
    file(WRITE ${hlBinDir}/${cmdGenFuncHeader}
        "gen=$1 func=$2 genArgs=$3 feat=$4;"
		"[ -f \"${hlHeaderExportDir}/$func.h\" ] && exit 0;"
		"echo generating $func.h with args=\\'\${genArgs}\\' feat=\\'\${feat}\\'; "
		"[ -f \${gen}_gen ] || (echo compiling \${gen}_gen ... && ${compileGen}); "
		"./\${gen}_gen -g $gen -f $func -o ${hlTarget}${hlTargetDirSuffix} -e h target=${hlTarget}$feat $genArgs ${hlAutoSchedule} || rm \${gen}_gen; "
		"cmake -E copy_if_different \"${hlTarget}${hlTargetDirSuffix}/$func.h\" \"${hlHeaderExportDirT}\"; "
	)

    set(cmdGenFuncLib ${hlTarget}${hlTargetDirSuffix}/gen_func.sh)	
    file(WRITE ${hlBinDir}/${cmdGenFuncLib}
		"gen=$1 && func=$2 && genArgs=$3 && feat=$4 && "
		"echo generating $func for target \\'${hlTarget}\${feat}\\' with args \\'\${genArgs}\\' && " 
		"./\${gen}_gen -g $gen -f $func -o ${hlTarget}${hlTargetDirSuffix} -e h,static_library,cpp,html${hlExtraEmit} target=${hlTarget}$feat $genArgs ${hlAutoSchedule} || (echo \"generator failed ($?)\" 1>&2 && exit 1) && "
		"cmake -E copy_if_different \"${hlTarget}${hlTargetDirSuffix}/$func.h\" \"${hlHeaderExportDirT}\""
	)

    set(cmdGenGen ${hlTarget}${hlTargetDirSuffix}/gen_gen.sh)
    file(WRITE ${hlBinDir}/${cmdGenGen} "gen=$1 && ${compileGen}")
endif ()

if(WINDOWS_LINUX_SUBSYS)
	set(cmdGenFuncHeader "${SHELL_CMD} ${cmdGenFuncHeader}")
	set(cmdGenFuncLib "${SHELL_CMD} ${cmdGenFuncLib}")
	set(cmdGenGen "${SHELL_CMD} ${cmdGenGen}")
endif()




file(STRINGS ${HLPIPES_ROOT}/Functions.txt hlFunctions)
foreach (funcDesc IN LISTS hlFunctions)
    if (NOT ${funcDesc} STREQUAL "")
        string(REGEX MATCH "^[\t ]*([a-zA-Z0-9_]+)[\t ]+([a-zA-Z0-9_]+)[\t ]*(.*)" funcDesc2 ${funcDesc})
        if (NOT ${funcDesc2} STREQUAL "")
            set(func ${CMAKE_MATCH_2}.defdep)
            set(funcDescIs "")
            if (EXISTS ${hlBinDir}/${func})
                file(READ ${hlBinDir}/${func} funcDescIs)
            endif ()
            file(GENERATE OUTPUT "${hlBinDir}/${func}" CONTENT "${funcDesc}" CONDITION $<NOT:$<STREQUAL:${funcDesc},${funcDescIs}>>)
        endif ()
    endif ()
endforeach ()

if("${hlGeneratorSources}" STREQUAL "")
	message(FATAL_ERROR "No *._generator.cpp in ${HLPIPES_ROOT}")
endif()


foreach (source IN LISTS hlGeneratorSources)
    generator_tag(tag ${source})

    set(genBin ${hlBinDir}/${tag}_gen${HOSTDOTEXE})

    # find header deps
    execute_process(
            COMMAND ${clangCompiler} -std=c++11 -M ${source} -I${HALIDE_DIR}/include
            WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
            OUTPUT_VARIABLE sourceDeps
    )


    string(REPLACE "\\ " "|" sourceDeps "${sourceDeps}")
    string(REPLACE "  " "" sourceDeps "${sourceDeps}")
    string(REPLACE "\r" "" sourceDeps "${sourceDeps}")
    string(REPLACE "|" " " sourceDeps "${sourceDeps}")
    string(REPLACE "\\\n" ";" sourceDeps "${sourceDeps}")
    list(REMOVE_AT sourceDeps 0)
    #message( FATAL_ERROR "${sourceDeps}")

    #message(FATAL_ERROR ${genBin})
    add_custom_command(
            OUTPUT ${genBin}
            COMMAND ${cmdGenGen} ${tag}
            DEPENDS ${source} "${sourceDeps}" ${hlPipelineMiscSources}
            WORKING_DIRECTORY "${hlBinDir}"
            COMMENT "generating ${tag} generator '${cmdGenGen} ${tag}' in ${hlBinDir}"
    )

    set(objNoFuncDesc "1")
    set(genFuncHeaders) #"#include \"${tag}_.h\""

    foreach (funcDesc IN LISTS hlFunctions)
        if (NOT ${funcDesc} STREQUAL "")
            string(REGEX MATCH "^[\t ]*${tag}[\t ]+([a-zA-Z0-9_]+)[\t ]*(target=-[a-z0-9_-]+)?(.*)" funcDesc2 ${funcDesc}) #\\+s([a-zA-Z0-9_]+)\\s*(.*) \\s*
            if (NOT ${funcDesc2} STREQUAL "")
                set(objNoFuncDesc "") # dont generate with default params if function definition exists for this generator

                set(funcLib ${hlTargetDir}/${CMAKE_MATCH_1}${DOTLIB})
                set(funcHeader ${hlTargetDir}/${CMAKE_MATCH_1}.h)
                #set(funcHeaderExport ${hlHeaderExportDir}/${CMAKE_MATCH_1}.h)

                list(APPEND hlPipelineLibs ${funcLib})
                list(APPEND hlPipelineHeaders ${funcHeader})
                set(hlAllHeaders "${hlAllHeaders}\n#include \"${CMAKE_MATCH_1}.h\"")
                set(genFuncHeaders "${genFuncHeaders}\n#include \"${CMAKE_MATCH_1}.h\"")

                string(REPLACE " " "\t" genTargetFeatures "${CMAKE_MATCH_2}")
                if (NOT ${genTargetFeatures} STREQUAL "")
                    string(SUBSTRING ${genTargetFeatures} 7 -1 genTargetFeatures)
                endif ()

                #exploting cmake bug to no escape spaces #TODO!
                string(REPLACE " " "\t" genParams "${CMAKE_MATCH_3}")

                add_custom_command(
                        OUTPUT ${funcLib} ${funcHeader} # the header creates the dep!
                        COMMAND ${cmdGenFuncLib} ARGS ${tag} ${CMAKE_MATCH_1} \"${genParams}\" ${genTargetFeatures}
                        DEPENDS ${hlBinDir}/${CMAKE_MATCH_1}.defdep ${genBin} ${sourceDeps} ${hlPipelineMiscSources} # ${source}   #${hlGenToolCpp}
                        WORKING_DIRECTORY "${hlBinDir}"
                        COMMENT "running ${tag}->${CMAKE_MATCH_1}(${genParams}) Function generator"
                )

                add_custom_command(
                        TARGET hl_pipelines_headers PRE_BUILD
                        COMMAND ${cmdGenFuncHeader} ARGS ${tag} ${CMAKE_MATCH_1} \"${genParams}\" ${genTargetFeatures}
                        WORKING_DIRECTORY "${hlBinDir}"
                        COMMENT "${tag}->${CMAKE_MATCH_1}(${genParams}) Header generator"
                )

                message(STATUS "Registered Halide Function generator ${tag}->${CMAKE_MATCH_1}")
            endif ()
        endif ()
    endforeach ()

    file(WRITE "${hlHeaderExportDir}/${tag}_all.h" "${genFuncHeaders}")

    # do 2 things (if pipeline source changed)
    # 1. compile generator to host-executable generator
    # 2. run the generator

    set(objLib ${hlTargetDir}/${tag}${DOTLIB}) # we need this outside generators loop!
    set(objHeader ${hlTargetDir}/${tag}.h)

    if (objNoFuncDesc)

        list(APPEND hlPipelineLibs ${objLib})
        list(APPEND hlPipelineHeaders ${objHeader})
        set(hlAllHeaders "${hlAllHeaders}\n#include \"${tag}.h\"")

        add_custom_command(
                OUTPUT ${objLib} ${objHeader} # the header creates the dep!
                COMMAND ${cmdGenFuncLib} ARGS ${tag} ${tag} \"\" \"\"
                DEPENDS ${genBin} ${sourceDeps} ${hlPipelineMiscSources} # ${source}  #${hlGenToolCpp}
                WORKING_DIRECTORY "${hlBinDir}"
                COMMENT "running ${tag} generator"
        )

        add_custom_command(
                TARGET hl_pipelines_headers PRE_BUILD
                COMMAND ${cmdGenFuncHeader} ${tag} ${tag} \"\" \"\"
                WORKING_DIRECTORY "${hlBinDir}"
                COMMENT "${tag} Header generator ${cmdGenFuncHeader} ${tag} ${tag} \"\" \"\" in ${hlBinDir}"
        )

        message(STATUS "Registered Halide generator ${tag}")
    endif ()
endforeach ()

file(WRITE "${hlHeaderExportDir}/all.h" "${hlAllHeaders}")

#TODO this might fail, if the last generator have not been compiled yet. should either build here again or take the first executable
# Halide runtime only module
set(hlRtLib ${hlTargetDir}/hlrt${DOTLIB})
set(hlRtHeader ${hlTargetDir}/hlrt.h)
list(APPEND hlPipelineLibs ${hlRtLib})
list(APPEND hlPipelineHeaders ${hlRtHeader})
translate_paths(hlBinDirT "${hlBinDir}")
#add_custom_command(
#        OUTPUT ${hlRtLib} ${hlRtHeader}
#        COMMAND ${SHELL_CMD} ${SHELL_BIN_PROXY} ${hlBinDirT}/${tag}_gen -r hlrt -o ${hlTarget}${hlTargetDirSuffix} target=${hlTarget}-opengl-debug && echo "//dep-dummy" > ${hlRtHeader}
#        DEPENDS ${objLib} # ${objLib} will make the generator to compile before #${hlGenToolCpp}
#        WORKING_DIRECTORY "${hlBinDir}"
#        COMMENT "running Halide runtime generator (with ${objBin})"
#)

add_custom_command(
        OUTPUT ${hlRtLib} ${hlRtHeader}
        COMMAND ${cmdGenFuncLib} ARGS ${tag} ${tag} \"-r hlrt\" \"-opengl-profile-debug\"
        COMMAND echo "//dep-dummy" > ${hlRtHeader}
        DEPENDS ${objLib}
        WORKING_DIRECTORY "${hlBinDir}"
        COMMENT "running Halide runtime generator (with ${tag}) ${hlRtLib}"
)


# we put the header here for dependencies, it have to be the generated header files, not the exports!
#file(GLOB hlExportedHeaders ${hlHeaderExportDir}/*.h)
add_library(hl_pipelines STATIC ${hlGeneratorSources} ${hlGenDir}/dummy.c ${hlGenToolCpp} ${hlPipelineHeaders})
set_source_files_properties(${hlGenToolCpp} PROPERTIES HEADER_FILE_ONLY TRUE)
target_link_libraries(hl_pipelines ${hlPipelineLibs})
set_target_properties(hl_pipelines PROPERTIES LINKER_LANGUAGE CXX)

add_dependencies(hl_pipelines hl_pipelines_headers)


# show the pipeline sources in the IDE (nothing should link against this)
add_library(hl_pipeline_sources SHARED EXCLUDE_FROM_ALL ${hlPipelineMiscSources} ${hlPipelineMiscHeaders} ${HLPIPES_ROOT}/Functions.txt)
set_target_properties(hl_pipeline_sources PROPERTIES LINKER_LANGUAGE CXX)
set_source_files_properties(${hlGeneratorSources} PROPERTIES HEADER_FILE_ONLY TRUE)
set_source_files_properties(${hlPipelineMiscSources} PROPERTIES HEADER_FILE_ONLY TRUE)
set_source_files_properties(${HLPIPES_ROOT}/Functions.txt PROPERTIES HEADER_FILE_ONLY TRUE)
source_group(generators FILES ${hlGeneratorSources})
source_group(misc_headers FILES ${hlPipelineMiscHeaders})
include_directories(${HALIDE_DIR}/include)

