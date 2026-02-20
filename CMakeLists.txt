cmake_minimum_required(VERSION 3.16)
project(LR3 CXX)

set(CMAKE_CXX_STANDARD 17)

# Включаем флаги покрытия
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fprofile-arcs -ftest-coverage")

include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0
)
FetchContent_MakeAvailable(googletest)

add_executable(tests
    test/main.cpp
    test/FBT_test.cpp
    test/array_test.cpp
    test/stack_test.cpp
    test/queue_test.cpp
    test/doubleList_test.cpp
    test/singleList_test.cpp
    test/hashTable_test.cpp
)

target_link_libraries(tests gtest gtest_main)
target_include_directories(tests PRIVATE . test)

# Проверяем наличие lcov и genhtml
find_program(LCOV_PATH lcov)
find_program(GENHTML_PATH genhtml)

if(NOT LCOV_PATH OR NOT GENHTML_PATH)
    message(WARNING "lcov или genhtml не найдены! LCOV отчет не будет сгенерирован.")
endif()

# Цель для создания отчёта LCOV
add_custom_target(coverage-lcov
    COMMAND ${CMAKE_COMMAND} -E echo "=== Очистка старых данных покрытия ==="
    COMMAND ${LCOV_PATH} --directory ${CMAKE_BINARY_DIR} --zerocounters
    
    COMMAND ${CMAKE_COMMAND} -E echo "=== Запуск тестов ==="
    COMMAND ${CMAKE_BINARY_DIR}/tests
    
    COMMAND ${CMAKE_COMMAND} -E echo "=== Сбор данных LCOV ==="
    COMMAND ${LCOV_PATH} --directory ${CMAKE_BINARY_DIR} --capture --output-file ${CMAKE_BINARY_DIR}/coverage.info --ignore-errors mismatch,empty,unused
    
    COMMAND ${CMAKE_COMMAND} -E echo "=== Удаление сторонних файлов ==="
    COMMAND ${LCOV_PATH} --remove ${CMAKE_BINARY_DIR}/coverage.info "/usr/*" "*/test/*" "*/_deps/*" "*/googletest/*" --output-file ${CMAKE_BINARY_DIR}/coverage_filtered.info --ignore-errors unused,empty
    
    COMMAND ${CMAKE_COMMAND} -E echo "=== Генерация HTML отчёта ==="
    COMMAND ${GENHTML_PATH} ${CMAKE_BINARY_DIR}/coverage_filtered.info --output-directory ${CMAKE_BINARY_DIR}/coverage_lcov --ignore-errors unmapped
    
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS tests
    COMMENT "Generating LCOV code coverage report"
)

# Цель для быстрого открытия LCOV отчёта
add_custom_target(coverage-lcov-open
    COMMAND ${CMAKE_COMMAND} -E echo "Открываю LCOV отчёт в браузере..."
    COMMAND xdg-open ${CMAKE_BINARY_DIR}/coverage_lcov/index.html 2>/dev/null ||
            ${CMAKE_COMMAND} -E echo "Используйте: firefox ${CMAKE_BINARY_DIR}/coverage_lcov/index.html"
    
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS coverage-lcov
    COMMENT "Opening LCOV coverage report in browser"
)

# Дополнительная цель для текстовой сводки LCOV
add_custom_target(coverage-lcov-summary
    COMMAND ${CMAKE_COMMAND} -E echo "=== Сводка покрытия LCOV ==="
    COMMAND ${LCOV_PATH} --summary ${CMAKE_BINARY_DIR}/coverage_filtered.info 2>/dev/null || 
            ${CMAKE_COMMAND} -E echo "Сводка недоступна"
    
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS coverage-lcov
    COMMENT "Showing LCOV coverage summary"
)


# Проверяем наличие gcovr
find_program(GCOVR_PATH gcovr)
if(NOT GCOVR_PATH)
    message(WARNING "gcovr not found! Coverage reports will not be generated.")
endif()

# Цель для создания чистого отчёта
add_custom_target(coverage
    COMMAND ${CMAKE_COMMAND} -E echo "=== Очистка старых данных ==="
    COMMAND ${CMAKE_COMMAND} -E rm -rf coverage_report
    COMMAND ${CMAKE_COMMAND} -E make_directory coverage_report
    
    COMMAND ${CMAKE_COMMAND} -E echo "=== Запуск тестов ==="
    COMMAND ./tests
    
    COMMAND ${CMAKE_COMMAND} -E echo "=== Создание HTML отчёта ==="
    COMMAND ${CMAKE_COMMAND} -E env "PATH=$ENV{PATH}"
    COMMAND ${CMAKE_COMMAND} -E chdir ${CMAKE_BINARY_DIR}
    COMMAND ${GCOVR_PATH} --root ${CMAKE_SOURCE_DIR}
        --exclude=".*/test/.*"
        --exclude=".*/_deps/.*"
        --exclude=".*/googletest/.*"
        --html coverage_report/coverage.html
        --html-details
        --print-summary
    
    COMMAND ${CMAKE_COMMAND} -E echo ""
    COMMAND ${CMAKE_COMMAND} -E echo " ОТЧЁТ ПОКРЫТИЯ СОЗДАН"
    COMMAND ${CMAKE_COMMAND} -E echo " Откройте: ${CMAKE_BINARY_DIR}/coverage_report/coverage.html"
    
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS tests
    COMMENT "Generating code coverage report"
)

# gcovr --root .. --exclude=".*/test/.*" --exclude=".*/_deps/.*" --exclude=".*/googletest/.*" --html coverage_report/coverage.html --html-details --print-summaryДополнительная цель для создания текстового отчета
add_custom_target(coverage-txt
    COMMAND ${GCOVR_PATH} --root ${CMAKE_SOURCE_DIR}
        --exclude=".*/test/.*"
        --exclude=".*/_deps/.*"
        --exclude=".*/googletest/.*"
        --txt coverage.txt
        --print-summary
    
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS tests
    COMMENT "Generating text coverage report"
)

# Цель для открытия отчета в браузере
add_custom_target(coverage-open
    COMMAND ${CMAKE_COMMAND} -E echo "Открываю отчёт в браузере..."
    COMMAND ${CMAKE_COMMAND} -E env "BROWSER=xdg-open"
    COMMAND xdg-open coverage_report/coverage.html 2>/dev/null || 
            ${CMAKE_COMMAND} -E echo "Используйте: firefox coverage_report/coverage.html"
    
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS coverage
    COMMENT "Opening coverage report in browser"
)
