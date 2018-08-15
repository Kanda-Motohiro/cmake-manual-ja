以下は、 git tag v3.12.0, ２０１８年８月時点の、cmake-buildsystem(7) の、
kanda.motohiro@gmail.com による抄訳です。BSD 3-Clause のもとで公開します。
rst ファイルのまま github に置くため、原文とレイアウトは異なります。
HTML へのビルドは、CMakeCache.txt の、SPHINX_HTML:BOOL=ON にして、 make したら、
Utilities/Sphinx/html 以下にできました。

cmake-buildsystem(7)
********************

Introduction
============

CMake をベースとするビルドシステムは、高レベルの論理的ターゲットの集まりと
して構成されます。ターゲットのそれぞれは実行可能ファイルあるいはライブラリ
に対応します。あるいは、カスタムコマンドを含むカスタムターゲットのこともあります。
ターゲット間の依存関係は、ビルドシステムで表現され、ビルド順と、
変更があったときの再生成の規則を決めます。

Binary Targets
==============

実行可能ファイルとライブラリは、 :command:`add_executable` と 
:command:`add_library` コマンドで定義します。
この結果できるバイナリファイルは、目的のプラットフォームにとって適切な
プレフィックスとサフィックスを持ちます。
バイナリターゲットの間の依存関係は、 :command:`target_link_libraries` 
コマンドで表現します。

.. code-block:: cmake

  add_library(archive archive.cpp zip.cpp lzma.cpp)
  add_executable(zipapp zipapp.cpp)
  target_link_libraries(zipapp archive)

``archive`` は静的ライブラリとして定義されます。 -- 
``archive.cpp``, ``zip.cpp``, and ``lzma.cpp`` からコンパイルされる
オブジェクトのアーカイブです。
``zipapp`` は ``zipapp.cpp`` をコンパイルし、リンクすることでできる
実行可能ファイルとして定義されます。
``zipapp`` 実行可能ファイルをリンクする時に、``archive`` 静的ライブラリが
結合されます。

Binary Executables
------------------

:command:`add_executable` コマンドは、実行可能ファイルターゲットを
定義します。

.. code-block:: cmake

  add_executable(mytool mytool.cpp)

ビルド時に実行される規則を生成する  :command:`add_custom_command` のようなコマンドは、
``COMMAND`` 実行可能ファイルとして、 :prop_tgt:`EXECUTABLE <TYPE>` ターゲットを
透過的に使うことができます。
ビルドシステム規則は、そのコマンドを実行しようとする前に、実行可能ファイルがビルドされることを保証します。

Binary Library Types
--------------------

.. _`Normal Libraries`:

Normal Libraries
^^^^^^^^^^^^^^^^

型が指定されていないならば、 :command:`add_library`  コマンドは、
デフォルトで静的ライブラリを定義します。

.. code-block:: cmake

  add_library(archive SHARED archive.cpp zip.cpp lzma.cpp)

.. code-block:: cmake

  add_library(archive STATIC archive.cpp zip.cpp lzma.cpp)

:variable:`BUILD_SHARED_LIBS` 変数を有効にして、
:command:`add_library` がデフォルトで共有ライブラリをビルドするように
振る舞いを変えることができます。

ビルドシステムの定義全体での文脈では、特定のライブラリが ``SHARED`` or ``STATIC``
であることはほぼ、意識されません。-- コマンド、依存関係の指定、そして
他の API は、ライブラリの型に関係なく、同様にはたらきます。
``MODULE`` ライブラリ型は、それが普通はリンクされないという点で特別です。
-- それは、:command:`target_link_libraries` コマンドの右辺には使われません。
それは、ランタイムの技術を使って、プラグインとしてロードされる型です。
ライブラリが、unmanaged symbol を何も公開しないならば、
（例えば、Windows リソース DLL, C++/CLI DLL）そのライブラリが、``SHARED``
ライブラリでないことが必要です。CMake は、 ``SHARED`` ライブラリが
少なくても１つのシンボルを公開することを期待するためです。

.. code-block:: cmake

  add_library(archive MODULE 7z.cpp)

.. _`Apple Frameworks`:

Apple Frameworks
""""""""""""""""

A ``SHARED`` library may be marked with the :prop_tgt:`FRAMEWORK`
target property to create an OS X or iOS Framework Bundle.
The ``MACOSX_FRAMEWORK_IDENTIFIER`` sets ``CFBundleIdentifier`` key
and it uniquely identifies the bundle.

.. code-block:: cmake

  add_library(MyFramework SHARED MyFramework.cpp)
  set_target_properties(MyFramework PROPERTIES
    FRAMEWORK TRUE
    FRAMEWORK_VERSION A
    MACOSX_FRAMEWORK_IDENTIFIER org.cmake.MyFramework
  )

.. _`Object Libraries`:

Object Libraries
^^^^^^^^^^^^^^^^

``OBJECT`` ライブラリ型は、指定されたソースファイルをコンパイルした結果できる
オブジェクトファイルの、アーカイブではない集まりを定義します。
これらのオブジェクトファイルは、他のターゲットのソース入力として
使うことができます。

.. code-block:: cmake

  add_library(archive OBJECT archive.cpp zip.cpp lzma.cpp)

  add_library(archiveExtras STATIC $<TARGET_OBJECTS:archive> extras.cpp)

  add_executable(test_exe $<TARGET_OBJECTS:archive> test.cpp)

他のターゲットでのリンク（あるいはアーカイブ）ステップは、そこで指定されたソースの他に、
オブジェクトファイルの集まりを使います。

あるいは、オブジェクトライブラリを、他のターゲットにリンクすることもできます。

.. code-block:: cmake

  add_library(archive OBJECT archive.cpp zip.cpp lzma.cpp)

  add_library(archiveExtras STATIC extras.cpp)
  target_link_libraries(archiveExtras PUBLIC archive)

  add_executable(test_exe test.cpp)
  target_link_libraries(test_exe archive)

他のターゲットでのリンク（あるいはアーカイブ）ステップは、オブジェクトライブラリ
にあるオブジェクトファイルを使い、それは *directly* にリンクされます。さらに、
前記、他のターゲットのソースをコンパイルする時に、オブジェクトライブラリの使用要件は順守されます。
さらに、この使用要件は、前記、他のターゲットが依存するものへと推移的に伝搬します。

:command:`add_custom_command(TARGET)` コマンドシグネチャにおいて、オブジェクト
ライブラリを、``TARGET`` として使うことはできません。
しかし、 ``$<TARGET_OBJECTS:objlib>`` を使って、オブジェクトのリストを、
 :command:`add_custom_command(OUTPUT)`
or :command:`file(GENERATE)` 
で使うことができます。

Build Specification and Usage Requirements
==========================================

:command:`target_include_directories`, :command:`target_compile_definitions`
and :command:`target_compile_options` コマンドは、バイナリターゲットのビルド指定と
使用要件を指定します。これらのコマンドはそれぞれ、
:prop_tgt:`INCLUDE_DIRECTORIES`, :prop_tgt:`COMPILE_DEFINITIONS` and
:prop_tgt:`COMPILE_OPTIONS` ターゲット属性と／あるいは
:prop_tgt:`INTERFACE_INCLUDE_DIRECTORIES`, :prop_tgt:`INTERFACE_COMPILE_DEFINITIONS`
and :prop_tgt:`INTERFACE_COMPILE_OPTIONS` ターゲット属性を設定します。

これらのコマンドは、 ``PRIVATE``, ``PUBLIC`` and ``INTERFACE`` モードを持ちます。
``PRIVATE`` モードは、 ``INTERFACE_`` がついていないターゲット属性だけを設定し、
``INTERFACE`` モードは、 ``INTERFACE_`` がついている方だけを設定します。
``PUBLIC`` モードは、指定されたターゲット属性の、両方の変種を設定します。
一つのコマンドで、これらキーワードを複数回使ってもかまいません。

.. code-block:: cmake

  target_compile_definitions(archive
    PRIVATE BUILDING_WITH_LZMA
    INTERFACE USING_ARCHIVE_LIB
  )

使用要件は、ダウンストリームが、特定の :prop_tgt:`COMPILE_OPTIONS` or
:prop_tgt:`COMPILE_DEFINITIONS` などを使うようにする便利な方法として設計されたのではありません。
属性の内容は、推奨や便宜ではなく、 **要件** でなくてはいけません。

再配布のためのパッケージを作る時に使用要件を指定する時に注意しなくてはいけないことについての議論は、
:manual:`cmake-packages(7)` マニュアルの :ref:`Creating Relocatable Packages`
セクションを参照ください。

Target Properties
-----------------

バイナリターゲットのソースファイルをコンパイルする時に、
:prop_tgt:`INCLUDE_DIRECTORIES`,
:prop_tgt:`COMPILE_DEFINITIONS` and :prop_tgt:`COMPILE_OPTIONS` ターゲット属性
の内容が適切に使われます。

:prop_tgt:`INCLUDE_DIRECTORIES` 内のエントリは、コンパイル行中に、
``-I`` or ``-isystem`` プレフィックスをつけて、属性値に現れる順番に
追加されます。

:prop_tgt:`COMPILE_DEFINITIONS` 内のエントリは、コンパイル行中に、
``-D`` or ``/D`` プレフィックスをつけて、追加されます。順序は不定です。
``SHARED`` and ``MODULE`` ライブラリターゲットの場合、特別に便宜を図るため、
:prop_tgt:`DEFINE_SYMBOL` ターゲット属性も、コンパイル定義として追加されます。

:prop_tgt:`COMPILE_OPTIONS` 内のエントリは、シェルエスケープされて、
属性値に現れる順番に追加されます

:prop_tgt:`POSITION_INDEPENDENT_CODE` のように、
特別な扱いが必要なコンパイルオプションがあります。

prop_tgt:`INTERFACE_INCLUDE_DIRECTORIES`,
:prop_tgt:`INTERFACE_COMPILE_DEFINITIONS` and
:prop_tgt:`INTERFACE_COMPILE_OPTIONS` 
ターゲット属性の内容は、*Usage Requirements* 、使用要件、です。
-- それは、それが指定されているターゲットの利用者が、正しくコンパイルし、ターゲットにリンクするために、
使わなくてはいけない内容を指定します。 
全てのバイナリターゲットで、 :command:`target_link_libraries` 
コマンドに指定された全てのターゲットにある ``INTERFACE_`` 属性の全ての内容が使われます。

.. code-block:: cmake

  set(srcs archive.cpp zip.cpp)
  if (LZMA_FOUND)
    list(APPEND srcs lzma.cpp)
  endif()
  add_library(archive SHARED ${srcs})
  if (LZMA_FOUND)
    # The archive library sources are compiled with -DBUILDING_WITH_LZMA
    target_compile_definitions(archive PRIVATE BUILDING_WITH_LZMA)
  endif()
  target_compile_definitions(archive INTERFACE USING_ARCHIVE_LIB)

  add_executable(consumer)
  # Link consumer to archive and consume its usage requirements. The consumer
  # executable sources are compiled with -DUSING_ARCHIVE_LIB.
  target_link_libraries(consumer archive)

ソースディレクトリと対応するビルドディレクトリが :prop_tgt:`INCLUDE_DIRECTORIES`
に加えられることを要求するのは一般的なので、
:variable:`CMAKE_INCLUDE_CURRENT_DIR`  変数を有効にして、全てのターゲットの 
:prop_tgt:`INCLUDE_DIRECTORIES` に、対応するディレクトリを加えることができます。
:variable:`CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE` 変数を有効にして、全てのターゲットの 
:prop_tgt:`INTERFACE_INCLUDE_DIRECTORIES` に、対応するディレクトリを加えることができます。
これは、 :command:`target_link_libraries` コマンドを、複数の異なるディレクトリに
あるターゲットに対して使う時に便利です。

.. _`Target Usage Requirements`:

Transitive Usage Requirements
-----------------------------

ターゲットの使用要件は、それが依存しているものに推移的に伝搬することがあります。
:command:`target_link_libraries` コマンドは、``PRIVATE``,
``INTERFACE`` and ``PUBLIC`` キーワードを持ち、伝搬を制御します。

.. code-block:: cmake

  add_library(archive archive.cpp)
  target_compile_definitions(archive INTERFACE USING_ARCHIVE_LIB)

  add_library(serialization serialization.cpp)
  target_compile_definitions(serialization INTERFACE USING_SERIALIZATION_LIB)

  add_library(archiveExtras extras.cpp)
  target_link_libraries(archiveExtras PUBLIC archive)
  target_link_libraries(archiveExtras PRIVATE serialization)
  # archiveExtras is compiled with -DUSING_ARCHIVE_LIB
  # and -DUSING_SERIALIZATION_LIB

  add_executable(consumer consumer.cpp)
  # consumer is compiled with -DUSING_ARCHIVE_LIB
  target_link_libraries(consumer archiveExtras)

``archive`` は、 ``archiveExtras`` の、 ``PUBLIC`` 依存関係ですから、
その使用要件は ``consumer`` にも伝搬されます。
``serialization`` は、 ``archiveExtras`` の、 ``PRIVATE`` 依存関係ですから、
その使用要件は ``consumer`` には伝搬されません。

一般的に、依存関係は、それが、ヘッダファイルにおいてではなく、そのライブラリの実装で使われるだけならば、
``PRIVATE`` キーワードを持つ :command:`target_link_libraries` を使って指定される
べきです。
もし依存関係がさらに、そのライブラリのヘッダファイルにおいても使われるならば（例えば、クラス継承）、
それは ``PUBLIC`` 依存関係として指定されるべきです。
ライブラリの実装で使われず、ヘッダファイルでだけ使われる依存関係は、 ``INTERFACE`` 依存関係
として指定されるべきです。
:command:`target_link_libraries` コマンドは、キーワードを複数使って呼ぶこともできます。

.. code-block:: cmake

  target_link_libraries(archiveExtras
    PUBLIC archive
    PRIVATE serialization
  )

依存しているもののターゲット属性にある ``INTERFACE_`` 変種を読んで、
その値を、オペランドの ``INTERFACE_`` でない変種に追加することによって伝搬されます。
例えば、依存しているものの :prop_tgt:`INTERFACE_INCLUDE_DIRECTORIES` 
が読まれ、オペランドの :prop_tgt:`INCLUDE_DIRECTORIES` に加えられます。
順序が重要で、維持されている時に、 :command:`target_link_libraries` 
呼び出しの結果作られる順序が、正しくコンパイルしない時は、
適切なコマンドを使って、属性を直接設定して、順序を変更できます。

例えば、ターゲットにリンクされるライブラリが、 ``lib1`` ``lib2`` ``lib3`` の順序で
指定されなくてはならず、インクルードディレクトリが ``lib3`` ``lib1`` ``lib2`` 
の順序で指定されなくてはいけないならば：

.. code-block:: cmake

  target_link_libraries(myExe lib1 lib2 lib3)
  target_include_directories(myExe
    PRIVATE $<TARGET_PROPERTY:lib3,INTERFACE_INCLUDE_DIRECTORIES>)

:command:`install(EXPORT)` コマンドを使って、インストール時にエクスポートされるターゲット
に対する使用要件を指定するときには、注意が必要です。詳しくは、 :ref:`Creating Packages` を参照ください。

.. _`Compatible Interface Properties`:

Compatible Interface Properties
-------------------------------

ターゲット属性によっては、ターゲットと依存するものそれぞれのインタフェースが
互換でなければいけないものがあります。
例えば、 :prop_tgt:`POSITION_INDEPENDENT_CODE`  ターゲット属性は、
ターゲットが位置非依存コードとしてコンパイルされるかどうかを指定するブール値を取ります。
それは、プラットフォーム固有の結果をもたらします。
さらに、ターゲットは、 :prop_tgt:`INTERFACE_POSITION_INDEPENDENT_CODE`
使用要件を指定して、消費者が位置非依存コードとしてコンパイルされなくてはいけないことを
指定することができます。

.. code-block:: cmake

  add_executable(exe1 exe1.cpp)
  set_property(TARGET exe1 PROPERTY POSITION_INDEPENDENT_CODE ON)

  add_library(lib1 SHARED lib1.cpp)
  set_property(TARGET lib1 PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE ON)

  add_executable(exe2 exe2.cpp)
  target_link_libraries(exe2 lib1)

ここで、 ``exe1`` and ``exe2`` の両方は、位置非依存コードとしてコンパイルされます。
``lib1`` も、位置非依存コードとしてコンパイルされます。それが、 ``SHARED`` ライブラリ
のデフォルト設定だからです。
依存関係が、競合し、互換でない要件を持つなら、 :manual:`cmake(1)` は、診断メッセージを出します。

.. code-block:: cmake

  add_library(lib1 SHARED lib1.cpp)
  set_property(TARGET lib1 PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE ON)

  add_library(lib2 SHARED lib2.cpp)
  set_property(TARGET lib2 PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE OFF)

  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 lib1)
  set_property(TARGET exe1 PROPERTY POSITION_INDEPENDENT_CODE OFF)

  add_executable(exe2 exe2.cpp)
  target_link_libraries(exe2 lib1 lib2)

The ``lib1`` requirement ``INTERFACE_POSITION_INDEPENDENT_CODE`` is not
"compatible" with the ``POSITION_INDEPENDENT_CODE`` property of the ``exe1``
target.  The library requires that consumers are built as
position-independent-code, while the executable specifies to not built as
position-independent-code, so a diagnostic is issued.

The ``lib1`` and ``lib2`` requirements are not "compatible".  One of them
requires that consumers are built as position-independent-code, while
the other requires that consumers are not built as position-independent-code.
Because ``exe2`` links to both and they are in conflict, a diagnostic is
issued.

To be "compatible", the :prop_tgt:`POSITION_INDEPENDENT_CODE` property,
if set must be either the same, in a boolean sense, as the
:prop_tgt:`INTERFACE_POSITION_INDEPENDENT_CODE` property of all transitively
specified dependencies on which that property is set.

This property of "compatible interface requirement" may be extended to other
properties by specifying the property in the content of the
:prop_tgt:`COMPATIBLE_INTERFACE_BOOL` target property.  Each specified property
must be compatible between the consuming target and the corresponding property
with an ``INTERFACE_`` prefix from each dependency:

.. code-block:: cmake

  add_library(lib1Version2 SHARED lib1_v2.cpp)
  set_property(TARGET lib1Version2 PROPERTY INTERFACE_CUSTOM_PROP ON)
  set_property(TARGET lib1Version2 APPEND PROPERTY
    COMPATIBLE_INTERFACE_BOOL CUSTOM_PROP
  )

  add_library(lib1Version3 SHARED lib1_v3.cpp)
  set_property(TARGET lib1Version3 PROPERTY INTERFACE_CUSTOM_PROP OFF)

  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 lib1Version2) # CUSTOM_PROP will be ON

  add_executable(exe2 exe2.cpp)
  target_link_libraries(exe2 lib1Version2 lib1Version3) # Diagnostic

Non-boolean properties may also participate in "compatible interface"
computations.  Properties specified in the
:prop_tgt:`COMPATIBLE_INTERFACE_STRING`
property must be either unspecified or compare to the same string among
all transitively specified dependencies. This can be useful to ensure
that multiple incompatible versions of a library are not linked together
through transitive requirements of a target:

.. code-block:: cmake

  add_library(lib1Version2 SHARED lib1_v2.cpp)
  set_property(TARGET lib1Version2 PROPERTY INTERFACE_LIB_VERSION 2)
  set_property(TARGET lib1Version2 APPEND PROPERTY
    COMPATIBLE_INTERFACE_STRING LIB_VERSION
  )

  add_library(lib1Version3 SHARED lib1_v3.cpp)
  set_property(TARGET lib1Version3 PROPERTY INTERFACE_LIB_VERSION 3)

  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 lib1Version2) # LIB_VERSION will be "2"

  add_executable(exe2 exe2.cpp)
  target_link_libraries(exe2 lib1Version2 lib1Version3) # Diagnostic

The :prop_tgt:`COMPATIBLE_INTERFACE_NUMBER_MAX` target property specifies
that content will be evaluated numerically and the maximum number among all
specified will be calculated:

.. code-block:: cmake

  add_library(lib1Version2 SHARED lib1_v2.cpp)
  set_property(TARGET lib1Version2 PROPERTY INTERFACE_CONTAINER_SIZE_REQUIRED 200)
  set_property(TARGET lib1Version2 APPEND PROPERTY
    COMPATIBLE_INTERFACE_NUMBER_MAX CONTAINER_SIZE_REQUIRED
  )

  add_library(lib1Version3 SHARED lib1_v3.cpp)
  set_property(TARGET lib1Version3 PROPERTY INTERFACE_CONTAINER_SIZE_REQUIRED 1000)

  add_executable(exe1 exe1.cpp)
  # CONTAINER_SIZE_REQUIRED will be "200"
  target_link_libraries(exe1 lib1Version2)

  add_executable(exe2 exe2.cpp)
  # CONTAINER_SIZE_REQUIRED will be "1000"
  target_link_libraries(exe2 lib1Version2 lib1Version3)

Similarly, the :prop_tgt:`COMPATIBLE_INTERFACE_NUMBER_MIN` may be used to
calculate the numeric minimum value for a property from dependencies.

Each calculated "compatible" property value may be read in the consumer at
generate-time using generator expressions.

Note that for each dependee, the set of properties specified in each
compatible interface property must not intersect with the set specified in
any of the other properties.

Property Origin Debugging
-------------------------

ビルド指定は依存関係で決まることがあるため、ターゲットを作るコードと、ビルド指定を設定する
役割を持つコードが離れていると、コードを理解するのがより困難になることがあります。
:manual:`cmake(1)` は、依存関係によって決定されることのある属性の内容の起源を表示するデバッグ機能を提供します。
デバッグできる属性の一覧は、 :variable:`CMAKE_DEBUG_TARGET_PROPERTIES` 
変数の文書にあります。

.. code-block:: cmake

  set(CMAKE_DEBUG_TARGET_PROPERTIES
    INCLUDE_DIRECTORIES
    COMPILE_DEFINITIONS
    POSITION_INDEPENDENT_CODE
    CONTAINER_SIZE_REQUIRED
    LIB_VERSION
  )
  add_executable(exe1 exe1.cpp)

:prop_tgt:`COMPATIBLE_INTERFACE_BOOL` or
:prop_tgt:`COMPATIBLE_INTERFACE_STRING` にリストされた属性の場合、デバッグ出力は
どのターゲットがその属性を設定する役割を果たしたか、そして、その属性を定義した依存関係が他にあったかを示します。
:prop_tgt:`COMPATIBLE_INTERFACE_NUMBER_MAX` and
:prop_tgt:`COMPATIBLE_INTERFACE_NUMBER_MIN` の場合、デバッグ出力は
それぞれの依存関係に由来する属性値と、その値が新しい限界値を決めたかを示します。

Build Specification with Generator Expressions
----------------------------------------------

ビルド指定は、条件によって決まる、あるいは、生成時までわからない内容を持つ、
:manual:`generator expressions <cmake-generator-expressions(7)>` 
を使うことができます。例えば、計算される、属性の "compatible" 値は、
``TARGET_PROPERTY`` 式で読むことができます。

.. code-block:: cmake

  add_library(lib1Version2 SHARED lib1_v2.cpp)
  set_property(TARGET lib1Version2 PROPERTY
    INTERFACE_CONTAINER_SIZE_REQUIRED 200)
  set_property(TARGET lib1Version2 APPEND PROPERTY
    COMPATIBLE_INTERFACE_NUMBER_MAX CONTAINER_SIZE_REQUIRED
  )

  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 lib1Version2)
  target_compile_definitions(exe1 PRIVATE
      CONTAINER_SIZE=$<TARGET_PROPERTY:CONTAINER_SIZE_REQUIRED>
  )

この場合、 ``exe1`` ソースファイルは、 ``-DCONTAINER_SIZE=200`` 付きで
コンパイルされます。

Configuration によって決まるビルド指定は、``CONFIG`` generator 式を使って設定できます。

.. code-block:: cmake

  target_compile_definitions(exe1 PRIVATE
      $<$<CONFIG:Debug>:DEBUG_BUILD>
  )

``CONFIG`` パラメタは、ビルドされている configuration と、大文字小文字の違いを無視して
比較されます。 :prop_tgt:`IMPORTED` ターゲットがある場合、
:prop_tgt:`MAP_IMPORTED_CONFIG_DEBUG <MAP_IMPORTED_CONFIG_<CONFIG>>`
の内容も、評価されます。

:manual:`cmake(1)`  が生成するビルドシステムによっては、あらかじめ決められた
ビルド configuration が、 :variable:`CMAKE_BUILD_TYPE` 変数に設定されていることが
あります。Visual Studio and Xcode のような IDE のためのビルドシステムは、
ビルド configuration と無関係に生成されます。そして、実際の ビルド configuration 
は、ビルド時までわかりません。なので、以下のようなコードは、

.. code-block:: cmake

  string(TOLOWER ${CMAKE_BUILD_TYPE} _type)
  if (_type STREQUAL debug)
    target_compile_definitions(exe1 PRIVATE DEBUG_BUILD)
  endif()

``Makefile`` ベースと ``Ninja`` generator では動くかもしれませんが、
IDE generator には移植可能ではありません。
このようなコードでは、configuration マッピングは期待できないので、使うべきではありません。

単項の ``TARGET_PROPERTY`` generator 式と ``TARGET_POLICY`` generator 式は、
それを消費するターゲットのコンテキストで評価されます。
これは、使用要件の指定は、消費者によって異なって評価されることがあることを意味します。

.. code-block:: cmake

  add_library(lib1 lib1.cpp)
  target_compile_definitions(lib1 INTERFACE
    $<$<STREQUAL:$<TARGET_PROPERTY:TYPE>,EXECUTABLE>:LIB1_WITH_EXE>
    $<$<STREQUAL:$<TARGET_PROPERTY:TYPE>,SHARED_LIBRARY>:LIB1_WITH_SHARED_LIB>
    $<$<TARGET_POLICY:CMP0041>:CONSUMER_CMP0041_NEW>
  )

  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 lib1)

  cmake_policy(SET CMP0041 NEW)

  add_library(shared_lib shared_lib.cpp)
  target_link_libraries(shared_lib lib1)

``exe1`` 実行可能ファイルは、 ``-DLIB1_WITH_EXE`` 付きでコンパイルされ、
``shared_lib``  共有ライブラリは  ``-DLIB1_WITH_SHARED_LIB``
and ``-DCONSUMER_CMP0041_NEW`` 付きでコンパイルされます。
``shared_lib`` ターゲットが作られた時に、:policy:`CMP0041` ポリシーは ``NEW`` 
だからです。

``BUILD_INTERFACE`` 式は、同じビルドシステムにあるターゲットによって消費される時と、
:command:`export` コマンドを使ってビルドディレクトリにエクスポートされるターゲットに
よって消費される時にだけ使われる要件をラップします。
``INSTALL_INTERFACE`` 式は、 :command:`install(EXPORT)`  コマンドによって
インストールされエクスポートされたターゲットよって消費される時にだけ使われる
要件をラップします。

.. code-block:: cmake

  add_library(ClimbingStats climbingstats.cpp)
  target_compile_definitions(ClimbingStats INTERFACE
    $<BUILD_INTERFACE:ClimbingStats_FROM_BUILD_LOCATION>
    $<INSTALL_INTERFACE:ClimbingStats_FROM_INSTALLED_LOCATION>
  )
  install(TARGETS ClimbingStats EXPORT libExport ${InstallArgs})
  install(EXPORT libExport NAMESPACE Upstream::
          DESTINATION lib/cmake/ClimbingStats)
  export(EXPORT libExport NAMESPACE Upstream::)

  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 ClimbingStats)

この場合、 ``exe1`` 実行可能ファイルファイルは、
``-DClimbingStats_FROM_BUILD_LOCATION`` 付きでコンパイルされます。
エクスポートするコマンドは、 ``INSTALL_INTERFACE`` あるいは 
``BUILD_INTERFACE`` が除かれ、 ``*_INTERFACE`` マーカーが取り除かれた
:prop_tgt:`IMPORTED` ターゲットを生成します。
``ClimbingStats`` パッケージを消費する独立したプロジェクトは、以下を含むでしょう：

.. code-block:: cmake

  find_package(ClimbingStats REQUIRED)

  add_executable(Downstream main.cpp)
  target_link_libraries(Downstream Upstream::ClimbingStats)

``ClimbingStats`` パッケージがビルド位置から使われるか、インストール位置から使われるかによって、
``Downstream`` ターゲットは、 
``-DClimbingStats_FROM_BUILD_LOCATION`` あるいは
``-DClimbingStats_FROM_INSTALL_LOCATION``
のどちらか付きでコンパイルされます。
パッケージとエクスポートについて詳しくは、:manual:`cmake-packages(7)` マニュアルを
参照ください。

.. _`Include Directories and Usage Requirements`:

Include Directories and Usage Requirements
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

インクルードディレクトリは、使用要件として指定する時と、generator 式で使う時には、いくらかの特殊な考慮が必要です。
:command:`target_include_directories` コマンドは、インクルードディレクトリの相対と絶対の両方の指定が可能です。

.. code-block:: cmake

  add_library(lib1 lib1.cpp)
  target_include_directories(lib1 PRIVATE
    /absolute/path
    relative/path
  )

相対パスは、そのコマンドが現れたソースディレクトリに相対的に評価されます。
相対パスは、 :prop_tgt:`IMPORTED` ターゲットの
:prop_tgt:`INTERFACE_INCLUDE_DIRECTORIES` には使えません。

自明でない generator 式を使う時、 ``INSTALL_INTERFACE`` 式の引数中で、
``INSTALL_PREFIX`` 式を使うことができます。
それは、それを消費するプロジェクトによってインポートされた時の、インストールプレフィックスに展開される置換マーカーです。

インクルードディレクトリの使用要件は普通は、ビルドツリーとインストールツリーで異なります。
``BUILD_INTERFACE`` and ``INSTALL_INTERFACE`` generator 式を使って、その使用位置に基づく別々の使用要件を記述することができます。
``INSTALL_INTERFACE`` 式内で、相対パスを使うことができ、それは、インストールプレフィックスに相対的に評価されます。
例えば：

.. code-block:: cmake

  add_library(ClimbingStats climbingstats.cpp)
  target_include_directories(ClimbingStats INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/generated>
    $<INSTALL_INTERFACE:/absolute/path>
    $<INSTALL_INTERFACE:relative/path>
    $<INSTALL_INTERFACE:$<INSTALL_PREFIX>/$<CONFIG>/generated>
  )

インクルードディレクトリ使用要件に関して、２つの便宜的 API が提供されています。
:variable:`CMAKE_INCLUDE_CURRENT_DIR_IN_INTERFACE` 変数を有効にすると、
全ての影響を受けるターゲットに対して、以下と同じ効果があります。

.. code-block:: cmake

  set_property(TARGET tgt APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR};${CMAKE_CURRENT_BINARY_DIR}>
  )

インストールされたターゲットに対する便宜的 API は、:command:`install(TARGETS)` コマンドにおける、
``INCLUDES DESTINATION`` コンポーネントです。

.. code-block:: cmake

  install(TARGETS foo bar bat EXPORT tgts ${dest_args}
    INCLUDES DESTINATION include
  )
  install(EXPORT tgts ${other_args})
  install(FILES ${headers} DESTINATION include)

これは、:command:`install(EXPORT)` が生成するインストールされた :prop_tgt:`IMPORTED` 
ターゲットそれぞれに対して、
``${CMAKE_INSTALL_PREFIX}/include`` を、
:prop_tgt:`INTERFACE_INCLUDE_DIRECTORIES` に加えることと同じです。

When the :prop_tgt:`INTERFACE_INCLUDE_DIRECTORIES` of an
:ref:`imported target <Imported targets>` is consumed, the entries in the
property are treated as ``SYSTEM`` include directories, as if they were
listed in the :prop_tgt:`INTERFACE_SYSTEM_INCLUDE_DIRECTORIES` of the
dependency. This can result in omission of compiler warnings for headers
found in those directories.  This behavior for :ref:`imported targets` may
be controlled by setting the :prop_tgt:`NO_SYSTEM_FROM_IMPORTED` target
property on the *consumers* of imported targets.

If a binary target is linked transitively to a Mac OX framework, the
``Headers`` directory of the framework is also treated as a usage requirement.
This has the same effect as passing the framework directory as an include
directory.

Link Libraries and Generator Expressions
----------------------------------------

ビルド指定と同様に、generator 式の条件に、 :prop_tgt:`link libraries <LINK_LIBRARIES>` 
を指定することができます。しかし、使用要件の消費は、リンクされた依存関係の集まりに基づきますから、
リンク依存関係が「有向無循環グラフ」を作らなくてはいけないという余分の制限があります。
つまり、ターゲットへのリンクが、ターゲット属性の値に依存するなら、そのターゲット属性はリンクされた依存関係に依存してはいけません。

.. code-block:: cmake

  add_library(lib1 lib1.cpp)
  add_library(lib2 lib2.cpp)
  target_link_libraries(lib1 PUBLIC
    $<$<TARGET_PROPERTY:POSITION_INDEPENDENT_CODE>:lib2>
  )
  add_library(lib3 lib3.cpp)
  set_property(TARGET lib3 PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE ON)

  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 lib1 lib3)

``exe1`` ターゲットの :prop_tgt:`POSITION_INDEPENDENT_CODE` 属性値はリンクされるライブラリ (``lib3``) 
に依存し、``exe1`` をリンクするエッジは、同じ :prop_tgt:`POSITION_INDEPENDENT_CODE` によって決まるため、
以上の依存関係グラフは循環を持ちます。
:manual:`cmake(1)` はこの場合、診断メッセージを出します。

.. _`Output Artifacts`:

Output Artifacts
----------------

:command:`add_library` and :command:`add_executable` コマンドが作る
ビルドシステムターゲットは、バイナリ出力を作る規則を作ります。
バイナリの正確な出力場所は、生成時にしか決められません。それはビルド構成とリンク依存関係
のリンク言語などに依存するからです。
生成されるバイナリの名前と場所をアクセスするために、 ``TARGET_FILE``,
``TARGET_LINKER_FILE`` そして関連する式を使うことができます。
しかしこれらの式は、``OBJECT`` ライブラリには使えません。式が指す、そのライブラリが生成する、
単一のファイルというものは無いからです。

以下のセクションで詳しく述べるように、ターゲットが作成する出力結果は３種類あります。
それらの分類は、DLL プラットフォームと DLL でないプラットフォームで異なります。
全ての、 Windows ベースのシステムは、Cygwin を含めて、DLL プラットフォームです。

.. _`Runtime Output Artifacts`:

Runtime Output Artifacts
^^^^^^^^^^^^^^^^^^^^^^^^

ビルドシステムターゲットの *runtime* 出力結果は以下のどれかです。

* 実行可能ターゲットの実行可能ファイル (e.g. ``.exe``) 
  これは、 :command:`add_executable` コマンドで作られます。

* DLL プラットフォームでは：共有ライブラリターゲットの実行可能ファイル (e.g. ``.dll``)
  これは、 ``SHARED`` オプション付きの :command:`add_library` コマンドで作られます。

:prop_tgt:`RUNTIME_OUTPUT_DIRECTORY` and :prop_tgt:`RUNTIME_OUTPUT_NAME`
ターゲット属性を使って、実行時の、ビルドツリー内での出力結果の場所と名前を制御できます。

.. _`Library Output Artifacts`:

Library Output Artifacts
^^^^^^^^^^^^^^^^^^^^^^^^

ビルドシステムターゲットの  *library* 出力結果は以下のどれかです。

* モジュールライブラリターゲットのローダブルモジュールファイル (e.g. ``.dll`` or ``.so``) of a module
  これは、 ``MODULE`` オプション付きの :command:`add_library` コマンドで作られます。

* DLL でないプラットフォームでは： 共有ライブラリターゲットの共有ライブラリファイル (e.g. ``.so`` or ``.dylib``)
  これは、 ``SHARED`` オプション付きの :command:`add_library` コマンドで作られます。

The :prop_tgt:`LIBRARY_OUTPUT_DIRECTORY` and :prop_tgt:`LIBRARY_OUTPUT_NAME`
ターゲット属性を使って、実行時の、ビルドツリー内でのライブラリ出力結果の場所と名前を制御できます。

.. _`Archive Output Artifacts`:

Archive Output Artifacts
^^^^^^^^^^^^^^^^^^^^^^^^

ビルドシステムターゲットの *archive* 出力結果は以下のどれかです。

* 静的ライブラリターゲットの静的ライブラリファイル (e.g. ``.lib`` or ``.a``) 
  これは、 ``STATIC`` オプション付きの :command:`add_library` コマンドで作られます。

* DLL プラットフォーム： ``SHARED`` オプションのある、 :command:`add_library` 
  コマンドで作成された共用ライブラリターゲットのインポートライブラリファイル
  (e.g. ``.lib``)。このファイルは、そのライブラリが少なくても一つの unmanaged symbol
  をエクスポートする時に限って、存在する保証があります。

* DLL プラットフォーム： :command:`add_executable` の、
  :prop_tgt:`ENABLE_EXPORTS` ターゲット属性が設定されている時、それが作成した実行可能ファイルターゲットのインポートライブラリ
  (e.g. ``.lib``)。

:prop_tgt:`ARCHIVE_OUTPUT_DIRECTORY` and :prop_tgt:`ARCHIVE_OUTPUT_NAME`
ターゲット属性を使って、実行時の、ビルドツリー内でのアーカイブ出力結果の場所と名前を制御できます。

Directory-Scoped Commands
-------------------------

:command:`target_include_directories`,
:command:`target_compile_definitions` and
:command:`target_compile_options`  コマンドは、一度に一つのターゲット
にだけ、影響します。
:command:`add_compile_definitions`,
:command:`add_compile_options` and :command:`include_directories` 
は類似の機能を持ちますが、ターゲットスコープではなく、ディレクトリのスコープで
はたらきます。

Pseudo Targets
==============

あるターゲット型は、ビルドシステムの出力を表さず、外部の依存関係、別名、あるいは
それ以外のビルド出力ではない入力を表します。擬似ターゲットは生成されたビルドシステム内では表されません。

.. _`Imported Targets`:

Imported Targets
----------------

:prop_tgt:`IMPORTED`  ターゲットは既存の依存関係を表します。通常、そのような
ターゲットはアップストリームのパッケージによって定義され、変更不可能として
扱われるべきです。
:prop_tgt:`IMPORTED` ターゲットを宣言した後、そのターゲット属性は、
:command:`target_compile_definitions`, :command:`target_include_directories`,
:command:`target_compile_options` or :command:`target_link_libraries` 
のような、customary コマンドで変更できます。これは、他の通常ターゲットと同じです。

:prop_tgt:`IMPORTED` ターゲットは、
:prop_tgt:`INTERFACE_INCLUDE_DIRECTORIES`,
:prop_tgt:`INTERFACE_COMPILE_DEFINITIONS`,
:prop_tgt:`INTERFACE_COMPILE_OPTIONS`,
:prop_tgt:`INTERFACE_LINK_LIBRARIES`, and
:prop_tgt:`INTERFACE_POSITION_INDEPENDENT_CODE`
のような、バイナリターゲットと同じ使用要件を持つことができます。

:prop_tgt:`IMPORTED` ターゲットから、 :prop_tgt:`LOCATION` を読むこともできますが、
そうする理由はほぼありません。
:command:`add_custom_command`  のようなコマンドは、``COMMAND`` 実行可能ファイルとして、
:prop_tgt:`IMPORTED` :prop_tgt:`EXECUTABLE <TYPE>`  を透過的に使うことができます。

The scope of the definition of an :prop_tgt:`IMPORTED` target is the directory
where it was defined.  It may be accessed and used from subdirectories, but
not from parent directories or sibling directories.  The scope is similar to
the scope of a cmake variable.

It is also possible to define a ``GLOBAL`` :prop_tgt:`IMPORTED` target which is
accessible globally in the buildsystem.

See the :manual:`cmake-packages(7)` manual for more on creating packages
with :prop_tgt:`IMPORTED` targets.

.. _`Alias Targets`:

Alias Targets
-------------

``ALIAS`` ターゲットは、リードオンリーのコンテキストで、バイナリターゲットの名前として使うことのできる名前です。
``ALIAS`` ターゲットの主なユースケースは、ライブラリに付属する例あるいはユニットテスト実行可能ファイルです。
それは、同じビルドシステムの一部であったり、ユーザの構成によって別にビルドされることがあるでしょう。

.. code-block:: cmake

  add_library(lib1 lib1.cpp)
  install(TARGETS lib1 EXPORT lib1Export ${dest_args})
  install(EXPORT lib1Export NAMESPACE Upstream:: ${other_args})

  add_library(Upstream::lib1 ALIAS lib1)

In another directory, we can link unconditionally to the ``Upstream::lib1``
target, which may be an :prop_tgt:`IMPORTED` target from a package, or an
``ALIAS`` target if built as part of the same buildsystem.

.. code-block:: cmake

  if (NOT TARGET Upstream::lib1)
    find_package(lib1 REQUIRED)
  endif()
  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 Upstream::lib1)

``ALIAS`` targets are not mutable, installable or exportable.  They are
entirely local to the buildsystem description.  A name can be tested for
whether it is an ``ALIAS`` name by reading the :prop_tgt:`ALIASED_TARGET`
property from it:

.. code-block:: cmake

  get_target_property(_aliased Upstream::lib1 ALIASED_TARGET)
  if(_aliased)
    message(STATUS "The name Upstream::lib1 is an ALIAS for ${_aliased}.")
  endif()

.. _`Interface Libraries`:

Interface Libraries
-------------------

``INTERFACE`` ターゲットは、 :prop_tgt:`LOCATION` を持たず、変更可能です。
しかしそれ以外は、 :prop_tgt:`IMPORTED` ターゲットと似ています。

それは、以下のような使用要件を指定できます。
:prop_tgt:`INTERFACE_INCLUDE_DIRECTORIES`,
:prop_tgt:`INTERFACE_COMPILE_DEFINITIONS`,
:prop_tgt:`INTERFACE_COMPILE_OPTIONS`,
:prop_tgt:`INTERFACE_LINK_LIBRARIES`,
:prop_tgt:`INTERFACE_SOURCES`,
and :prop_tgt:`INTERFACE_POSITION_INDEPENDENT_CODE`.
``INTERFACE`` ライブラリに対しては、
:command:`target_include_directories`,
:command:`target_compile_definitions`, :command:`target_compile_options`,
:command:`target_sources`, and :command:`target_link_libraries` 
の、 ``INTERFACE`` モードだけを使うことができます。

``INTERFACE`` ライブラリの主なユースケースは、ヘッダだけのライブラリです。

.. code-block:: cmake

  add_library(Eigen INTERFACE)
  target_include_directories(Eigen INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    $<INSTALL_INTERFACE:include/Eigen>
  )

  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 Eigen)

ここで、 ``Eigen`` ターゲットからの使用要件は、コンパイル時に消費され、
使われますが、リンク時には何も影響を持ちません。

Another use-case is to employ an entirely target-focussed design for usage
requirements:

.. code-block:: cmake

  add_library(pic_on INTERFACE)
  set_property(TARGET pic_on PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE ON)
  add_library(pic_off INTERFACE)
  set_property(TARGET pic_off PROPERTY INTERFACE_POSITION_INDEPENDENT_CODE OFF)

  add_library(enable_rtti INTERFACE)
  target_compile_options(enable_rtti INTERFACE
    $<$<OR:$<COMPILER_ID:GNU>,$<COMPILER_ID:Clang>>:-rtti>
  )

  add_executable(exe1 exe1.cpp)
  target_link_libraries(exe1 pic_on enable_rtti)

This way, the build specification of ``exe1`` is expressed entirely as linked
targets, and the complexity of compiler-specific flags is encapsulated in an
``INTERFACE`` library target.

The properties permitted to be set on or read from an ``INTERFACE`` library
are:

* Properties matching ``INTERFACE_*``
* Built-in properties matching ``COMPATIBLE_INTERFACE_*``
* ``EXPORT_NAME``
* ``IMPORTED``
* ``NAME``
* Properties matching ``IMPORTED_LIBNAME_*``
* Properties matching ``MAP_IMPORTED_CONFIG_*``

``INTERFACE`` libraries may be installed and exported.  Any content they refer
to must be installed separately:

.. code-block:: cmake

  add_library(Eigen INTERFACE)
  target_include_directories(Eigen INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    $<INSTALL_INTERFACE:include/Eigen>
  )

  install(TARGETS Eigen EXPORT eigenExport)
  install(EXPORT eigenExport NAMESPACE Upstream::
    DESTINATION lib/cmake/Eigen
  )
  install(FILES
      ${CMAKE_CURRENT_SOURCE_DIR}/src/eigen.h
      ${CMAKE_CURRENT_SOURCE_DIR}/src/vector.h
      ${CMAKE_CURRENT_SOURCE_DIR}/src/matrix.h
    DESTINATION include/Eigen
  )
