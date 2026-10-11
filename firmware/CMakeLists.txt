# The following lines of boilerplate have to be in your project's
# CMakeLists in this exact order for cmake to work correctly
cmake_minimum_required(VERSION 3.5)
set(EXTRA_COMPONENT_DIRS "esp-idf-lib/components")

# ESP-IDF sets the app version from `git describe --dirty` only when CMake
# configures. A change to any tracked file, the index or the branch tip makes
# the next build configure again, so the version matches the tree it built.
execute_process(
	COMMAND git ls-files
	WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
	RESULT_VARIABLE git_ls_result
	OUTPUT_VARIABLE git_tracked
	OUTPUT_STRIP_TRAILING_WHITESPACE
	ERROR_QUIET)
execute_process(
	COMMAND git rev-parse --absolute-git-dir
	WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
	OUTPUT_VARIABLE git_dir
	OUTPUT_STRIP_TRAILING_WHITESPACE
	ERROR_QUIET)
execute_process(
	COMMAND git symbolic-ref -q HEAD
	WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
	OUTPUT_VARIABLE git_branch_ref
	OUTPUT_STRIP_TRAILING_WHITESPACE
	ERROR_QUIET)
if(git_ls_result EQUAL 0)
	string(REPLACE "\n" ";" git_tracked "${git_tracked}")
	set(version_inputs "")
	foreach(path ${git_tracked})
		list(APPEND version_inputs "${CMAKE_CURRENT_LIST_DIR}/${path}")
	endforeach()
	list(APPEND version_inputs "${git_dir}/HEAD" "${git_dir}/index" "${git_dir}/packed-refs")
	if(git_branch_ref)
		list(APPEND version_inputs "${git_dir}/${git_branch_ref}")
	endif()
	foreach(path ${version_inputs})
		# A submodule is listed as a directory, and a deleted file as a
		# missing path; neither can be a configure dependency.
		if(EXISTS "${path}" AND NOT IS_DIRECTORY "${path}")
			set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS "${path}")
		endif()
	endforeach()
endif()

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
get_filename_component(ProjectId ${CMAKE_CURRENT_LIST_DIR} NAME)
string(REPLACE " " "_" ProjectId ${ProjectId})
project(${ProjectId})
