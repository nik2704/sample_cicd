### Репозиторий содержит пример простого приложения с автоматизацией CI/CD
- #### Автоматическая сбора на Github (`release.yaml`)
- #### Настройки `CMakeLists.txt`
#
#
#### Настройки `CMakeLists.txt` с поддержкой модульного тестирования с помощью библиотеки Boost

1. **Минимальная версия CMake**:
   ```cmake
   cmake_minimum_required(VERSION 3.12)
   ```

2. **Настройка версии проекта**:
   ```cmake
   set(PATCH_VERSION "1" CACHE INTERNAL "Patch version")
   set(PROJECT_VERSION 0.0.${PATCH_VERSION})
   project(helloworld VERSION ${PROJECT_VERSION})
   ```
   Устанавливается версия проекта как `0.0.1`, где `PATCH_VERSION` можно менять для контроля версий.

3. **Опция для включения тестов с Boost**:
   ```cmake
   option(WITH_BOOST_TEST "Whether to build Boost test" ON)
   ```
   Эта опция включает или отключает поддержку тестов Boost.

4. **Файл конфигурации для версии**:
   ```cmake
   configure_file(version.h.in version.h)
   ```
   Создается файл `version.h` на основе шаблона `version.h.in` для хранения информации о версии.

5. **Создание исполняемого и библиотечного файлов**:
   ```cmake
   add_executable(helloworld_cli main.cpp)
   add_library(helloworld lib.cpp)
   ```
   Определяются целевые файлы: `helloworld_cli` (исполняемый файл) и `helloworld` (библиотека).

6. **Свойства целевых файлов**:
   ```cmake
   set_target_properties(helloworld_cli helloworld PROPERTIES
       CXX_STANDARD 14
       CXX_STANDARD_REQUIRED ON
   )
   ```
   Устанавливается стандарт C++14 для `helloworld_cli` и `helloworld`.

7. **Включение каталогов для библиотеки**:
   ```cmake
   target_include_directories(helloworld
       PRIVATE "${CMAKE_BINARY_DIR}"
   )
   ```
   `helloworld` включает каталоги заголовочных файлов из `CMAKE_BINARY_DIR`, где находится `version.h`.

8. **Связывание библиотек**:
   ```cmake
   target_link_libraries(helloworld_cli PRIVATE
       helloworld
   )
   ```
   Исполняемый файл `helloworld_cli` связывается с библиотекой `helloworld`.

9. **Добавление теста с Boost (при включенной опции)**:
   ```cmake
   if(WITH_BOOST_TEST)
       find_package(Boost COMPONENTS unit_test_framework REQUIRED)
       add_executable(test_version test_version.cpp)
       set_target_properties(test_version PROPERTIES
           CXX_STANDARD 14
           CXX_STANDARD_REQUIRED ON
           COMPILE_DEFINITIONS BOOST_TEST_DYN_LINK
           INCLUDE_DIRECTORIES ${Boost_INCLUDE_DIR}
       )
       target_link_libraries(test_version
           ${Boost_LIBRARIES}
           helloworld
       )
   endif()
   ```
   Если `WITH_BOOST_TEST` включен, то ищется библиотека Boost и создается исполняемый файл `test_version` для тестов.

10. **Настройки компилятора для предупреждений**:
    ```cmake
    if (MSVC)
        target_compile_options(... /W4 ...)
    else ()
        target_compile_options(... -Wall -Wextra -pedantic -Werror ...)
    endif()
    ```
    Задание уровней предупреждений для Visual Studio (`/W4`) и GCC/Clang (`-Wall -Wextra -pedantic -Werror`).

11. **Установка и упаковка**:
    ```cmake
    install(TARGETS helloworld_cli RUNTIME DESTINATION bin)
    set(CPACK_GENERATOR DEB)
    set(CPACK_PACKAGE_VERSION_MAJOR "${PROJECT_VERSION_MAJOR}")
    set(CPACK_PACKAGE_VERSION_MINOR "${PROJECT_VERSION_MINOR}")
    set(CPACK_PACKAGE_VERSION_PATCH "${PROJECT_VERSION_PATCH}")
    set(CPACK_PACKAGE_CONTACT example@example.com)
    include(CPack)
    ```
    Устанавливает `helloworld_cli` в папку `bin` и добавляет поддержку создания Debian-пакета (DEB).

12. **Настройка тестирования**:
    ```cmake
    if(WITH_BOOST_TEST)
        enable_testing()
        add_test(test_version test_version)
    endif()
    ```
    При включении Boost-тестов добавляется тест `test_version`, который может запускаться с помощью `ctest`.

#### Автоматическая сбора на Github (`release.yaml` — конфигурация для GitHub Actions)

1. **Триггер события**:
   ```yaml
   on:
     push:
       branches:
         - '*'
   ```
   Запускает workflow на GitHub Actions при пуше в любую ветку.

2. **Определение задания (job)**:
   ```yaml
   jobs:
     build:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v2
           with:
             submodules: true
   ```
   В job `build` указывается, что оно выполняется на `ubuntu-latest`, и осуществляется проверка исходного кода (с поддержкой подмодулей, если они есть).

3. **Установка зависимостей**:
   ```yaml
   - run: sudo apt-get update && sudo apt-get install libboost-test-dev -y
   ```
   Устанавливается пакет `libboost-test-dev`, необходимый для тестов с использованием Boost.

4. **Конфигурация CMake с использованием run number**:
   ```yaml
   - run: cmake . -DPATCH_VERSION=${{ github.run_number }} -DWITH_BOOST_TEST=ON
   ```
   Запускает CMake для конфигурации проекта, устанавливая `PATCH_VERSION` в значение текущего номера запуска (`github.run_number`) и активируя опцию `WITH_BOOST_TEST`.

5. **Сборка проекта**:
   ```yaml
   - run: cmake --build .
   ```
   Собирает проект с помощью CMake.

6. **Запуск тестов**:
   ```yaml
   - run: cmake --build . --target test
   ```
   Выполняет тесты (если они включены) через CMake.

7. **Создание пакета**:
   ```yaml
   - run: cmake --build . --target package
   ```
   Создает Debian-пакет (`.deb`) с помощью CPack.

8. **Создание релиза на GitHub**:
   ```yaml
   - name: Create Release
     id: create_release
     uses: actions/create-release@v1
     env:
       GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
     with:
       tag_name: ${{ github.run_number }}
       release_name: Release ${{ github.run_number }}
       draft: false
       prerelease: false
   ```
   Создает релиз на GitHub с уникальным тегом, равным текущему номеру запуска (`github.run_number`). Использует `GITHUB_TOKEN` для аутентификации.

9. **Загрузка созданного пакета как ассета релиза**:
   ```yaml
   - name: Upload Release Asset
     id: upload-release-asset
     uses: actions/upload-release-asset@v1
     env:
       GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
     with:
       upload_url: ${{ steps.create_release.outputs.upload_url }}
       asset_path: ./helloworld-0.0.${{ github.run_number }}-Linux.deb
       asset_name: helloworld-0.0.${{ github.run_number }}-Linux.deb
       asset_content_type: application/vnd.debian.binary-package
   ```
   Загружает сгенерированный Debian-пакет в качестве ассета для созданного релиза. Пакет получает имя вида `helloworld-0.0.<run_number>-Linux.deb`, а `asset_content_type` указывает на тип контента для Debian.

Таким образом, каждый пуш в любую ветку запускает сборку, выполнение тестов, создание пакета, создание релиза и загрузку пакета как ассета.
