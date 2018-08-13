以下は、 git tag v3.12.0, ２０１８年８月時点の、target_link_libraries の、
kanda.motohiro@gmail.com による抄訳です。BSD 3-Clause のもとで公開します。

target_link_libraries
---------------------

与えられたターゲットそして／あるいはそれが依存するものをリンクする時に使うライブラリあるいはフラグを指定します。
リンクされるライブラリターゲットの
:ref:`Usage requirements <Target Usage Requirements>`
使用要件は伝搬されます。
ターゲットが依存するものの使用要件は、それ自身のソースのコンパイルに影響します。

Overview
^^^^^^^^

このコマンドはいくつかのシグネチャを持ちます。それぞれは、以下のサブセクションで説明します。
全ては以下の一般形を持ちます::

  target_link_libraries(<target> ... <item>... ...)

指定された ``<target>`` は、
:command:`add_executable` or :command:`add_library` 
のようなコマンドに依って、現在ディレクトリに作られていなければいけません。
それは、 :ref:`ALIAS target <Alias Targets>` 別名ではいけません。
同じ ``<target>`` への複数の呼び出しは、呼び出し順に、アイテムを追加します。
それぞれの ``<item>``  は以下のいずれかです。

* **A library target name**: 生成されたリンク行は、ターゲットに関連づいた
  リンク可能ライブラリファイルの完全パスを持ちます。ビルドシステムは、ライブラリファイルが
  変更されたら、``<target>`` を再リンクする依存関係を持ちます。

  指定されたターゲットはプロジェクト内で :command:`add_library` あるいは
  :ref:`IMPORTED library <Imported Targets>` によって作られなくてはいけません。
  もしそれがプロジェクト内で作られたなら、自動的にビルドシステムに順序依存関係が追加されて、
  指定されたライブラリターゲットが ``<target>`` がリンクする前に最新であることが保証されます。

  もしインポートされたライブラリが、 :prop_tgt:`IMPORTED_NO_SONAME` ターゲット属性
  を設定されていたら、CMake は、完全パスを使わずに、リンカーに、ライブラリを探すように頼むことがあります。
  (例えば、 ``/usr/lib/libfoo.so`` は、 ``-lfoo`` になります)。

* **A full path to a library file**: 生成されたリンク行は、通常は、
  そのファイルへの完全パスを保持します。ビルドシステムは、ライブラリファイルが
  変更されたら、``<target>`` を再リンクする依存関係を持ちます。
  共用ライブラリが  ``SONAME`` フィールドを何も持たないことが検出された時など、
  CMake がリンカーにライブラリを探すように頼むことがあります。
  (例えば、 ``/usr/lib/libfoo.so`` は、 ``-lfoo`` になります)。
  それ以外の場合については、:policy:`CMP0060` ポリシーを参照ください。

  ライブラリファイルが Mac OSX フレームワークにあるなら、フレームワークの ``Headers`` 
  ディレクトリも、:ref:`usage requirement <Target Usage Requirements>` として
  処理されます。これは、フレームワークのディレクトリを、インクルードディレクトリとして与えるのと同じ効果を持ちます。

  VS 2010 以上の :ref:`Visual Studio Generators` では、``.targets`` で終わる
  ライブラリファイルは、MSBuild ターゲットファイルとして扱われ、生成されたプロジェクトファイルに
  インポートされます。これは、他の generator ではサポートされません。

* **A plain library name**: 生成されたリンク行は、
  リンカーに、ライブラリを探すように頼みます。
  (例えば、 ``foo`` は ``-lfoo`` or ``foo.lib`` になります)。

* **A link flag**: ``-l`` あるいは ``-framework`` 以外の、``-`` で始まるアイテム名は
  リンカーフラグとして扱われます。
  それらフラグは、推移的依存関係に関しては、他のライブラリリンクアイテムと同じように
  扱われることに注意ください。なので、一般的に、依存するものに伝搬しないものは、
  プライベートなリンクアイテムとしてだけ指定するのが安全です。
  ここで指定されたリンクフラグは、リンクコマンドで、リンクライブラリと同じ位置に挿入されます。
  これは、リンカーによっては、正しくありません。明示的にリンクフラグを加えるには、
  :prop_tgt:`LINK_OPTIONS` ターゲット属性か、 :command:`target_link_options` 
  コマンドを使ってください。そうすれば、フラグは、リンクコマンドで、ツールチェーンが決めたフラグ位置に置かれます。

* ``<item>`` の直前に置かれる、 ``debug``, ``optimized``, or ``general`` キーワード。
  それらキーワードの後にあるアイテムは、対応するビルド configuration でだけ使われます。
  ``debug`` キーワードは、``Debug`` configuration （あるいは、
  :prop_gbl:`DEBUG_CONFIGURATIONS` グローバル属性が設定されていたら、そこに指定
  される configuration 名）に対応します。
  ``optimized`` キーワードは、他の全ての configuration に対応します。
  ``general`` キーワードは全ての configuration に対応し、純粋にオプショナルです。
  :ref:`IMPORTED library targets <Imported Targets>` を作成し、リンクすることで、
  configuration 規則ごとにより高度な粒度を達成できます。

Items containing ``::``, such as ``Foo::Bar``, are assumed to be
:ref:`IMPORTED <Imported Targets>` or :ref:`ALIAS <Alias Targets>` library
target names and will cause an error if no such target exists.
See policy :policy:`CMP0028`.

Arguments to ``target_link_libraries`` may use "generator expressions"
with the syntax ``$<...>``.  Note however, that generator expressions
will not be used in OLD handling of :policy:`CMP0003` or :policy:`CMP0004`.
See the :manual:`cmake-generator-expressions(7)` manual for available
expressions.  See the :manual:`cmake-buildsystem(7)` manual for more on
defining buildsystem properties.

Libraries for a Target and/or its Dependents
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

  target_link_libraries(<target>
                        <PRIVATE|PUBLIC|INTERFACE> <item>...
                       [<PRIVATE|PUBLIC|INTERFACE> <item>...]...)

The ``PUBLIC``, ``PRIVATE`` and ``INTERFACE`` keywords can be used to
specify both the link dependencies and the link interface in one command.
Libraries and targets following ``PUBLIC`` are linked to, and are made
part of the link interface.  Libraries and targets following ``PRIVATE``
are linked to, but are not made part of the link interface.  Libraries
following ``INTERFACE`` are appended to the link interface and are not
used for linking ``<target>``.

Libraries for both a Target and its Dependents
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

  target_link_libraries(<target> <item>...)

Library dependencies are transitive by default with this signature.
When this target is linked into another target then the libraries
linked to this target will appear on the link line for the other
target too.  This transitive "link interface" is stored in the
:prop_tgt:`INTERFACE_LINK_LIBRARIES` target property and may be overridden
by setting the property directly.  When :policy:`CMP0022` is not set to
``NEW``, transitive linking is built in but may be overridden by the
:prop_tgt:`LINK_INTERFACE_LIBRARIES` property.  Calls to other signatures
of this command may set the property making any libraries linked
exclusively by this signature private.

Libraries for a Target and/or its Dependents (Legacy)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

  target_link_libraries(<target>
                        <LINK_PRIVATE|LINK_PUBLIC> <lib>...
                       [<LINK_PRIVATE|LINK_PUBLIC> <lib>...]...)

The ``LINK_PUBLIC`` and ``LINK_PRIVATE`` modes can be used to specify both
the link dependencies and the link interface in one command.

This signature is for compatibility only.  Prefer the ``PUBLIC`` or
``PRIVATE`` keywords instead.

Libraries and targets following ``LINK_PUBLIC`` are linked to, and are
made part of the :prop_tgt:`INTERFACE_LINK_LIBRARIES`.  If policy
:policy:`CMP0022` is not ``NEW``, they are also made part of the
:prop_tgt:`LINK_INTERFACE_LIBRARIES`.  Libraries and targets following
``LINK_PRIVATE`` are linked to, but are not made part of the
:prop_tgt:`INTERFACE_LINK_LIBRARIES` (or :prop_tgt:`LINK_INTERFACE_LIBRARIES`).

Libraries for Dependents Only (Legacy)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

  target_link_libraries(<target> LINK_INTERFACE_LIBRARIES <item>...)

The ``LINK_INTERFACE_LIBRARIES`` mode appends the libraries to the
:prop_tgt:`INTERFACE_LINK_LIBRARIES` target property instead of using them
for linking.  If policy :policy:`CMP0022` is not ``NEW``, then this mode
also appends libraries to the :prop_tgt:`LINK_INTERFACE_LIBRARIES` and its
per-configuration equivalent.

This signature is for compatibility only.  Prefer the ``INTERFACE`` mode
instead.

Libraries specified as ``debug`` are wrapped in a generator expression to
correspond to debug builds.  If policy :policy:`CMP0022` is
not ``NEW``, the libraries are also appended to the
:prop_tgt:`LINK_INTERFACE_LIBRARIES_DEBUG <LINK_INTERFACE_LIBRARIES_<CONFIG>>`
property (or to the properties corresponding to configurations listed in
the :prop_gbl:`DEBUG_CONFIGURATIONS` global property if it is set).
Libraries specified as ``optimized`` are appended to the
:prop_tgt:`INTERFACE_LINK_LIBRARIES` property.  If policy :policy:`CMP0022`
is not ``NEW``, they are also appended to the
:prop_tgt:`LINK_INTERFACE_LIBRARIES` property.  Libraries specified as
``general`` (or without any keyword) are treated as if specified for both
``debug`` and ``optimized``.

Linking Object Libraries
^^^^^^^^^^^^^^^^^^^^^^^^

:ref:`Object Libraries` may be used as the ``<target>`` (first) argument
of ``target_link_libraries`` to specify dependencies of their sources
on other libraries.  For example, the code

.. code-block:: cmake

  add_library(A SHARED a.c)
  target_compile_definitions(A PUBLIC A)

  add_library(obj OBJECT obj.c)
  target_compile_definitions(obj PUBLIC OBJ)
  target_link_libraries(obj PUBLIC A)

compiles ``obj.c`` with ``-DA -DOBJ`` and establishes usage requirements
for ``obj`` that propagate to its dependents.

Normal libraries and executables may link to :ref:`Object Libraries`
to get their objects and usage requirements.  Continuing the above
example, the code

.. code-block:: cmake

  add_library(B SHARED b.c)
  target_link_libraries(B PUBLIC obj)

compiles ``b.c`` with ``-DA -DOBJ``, creates shared library ``B``
with object files from ``b.c`` and ``obj.c``, and links ``B`` to ``A``.
Furthermore, the code

.. code-block:: cmake

  add_executable(main main.c)
  target_link_libraries(main B)

compiles ``main.c`` with ``-DA -DOBJ`` and links executable ``main``
to ``B`` and ``A``.  The object library's usage requirements are
propagated transitively through ``B``, but its object files are not.

:ref:`Object Libraries` may "link" to other object libraries to get
usage requirements, but since they do not have a link step nothing
is done with their object files.  Continuing from the above example,
the code:

.. code-block:: cmake

  add_library(obj2 OBJECT obj2.c)
  target_link_libraries(obj2 PUBLIC obj)

  add_executable(main2 main2.c)
  target_link_libraries(main2 obj2)

compiles ``obj2.c`` with ``-DA -DOBJ``, creates executable ``main2``
with object files from ``main2.c`` and ``obj2.c``, and links ``main2``
to ``A``.

In other words, when :ref:`Object Libraries` appear in a target's
:prop_tgt:`INTERFACE_LINK_LIBRARIES` property they will be
treated as :ref:`Interface Libraries`, but when they appear in
a target's :prop_tgt:`LINK_LIBRARIES` property their object files
will be included in the link too.

Cyclic Dependencies of Static Libraries
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The library dependency graph is normally acyclic (a DAG), but in the case
of mutually-dependent ``STATIC`` libraries CMake allows the graph to
contain cycles (strongly connected components).  When another target links
to one of the libraries, CMake repeats the entire connected component.
For example, the code

.. code-block:: cmake

  add_library(A STATIC a.c)
  add_library(B STATIC b.c)
  target_link_libraries(A B)
  target_link_libraries(B A)
  add_executable(main main.c)
  target_link_libraries(main A)

links ``main`` to ``A B A B``.  While one repetition is usually
sufficient, pathological object file and symbol arrangements can require
more.  One may handle such cases by using the
:prop_tgt:`LINK_INTERFACE_MULTIPLICITY` target property or by manually
repeating the component in the last ``target_link_libraries`` call.
However, if two archives are really so interdependent they should probably
be combined into a single archive, perhaps by using :ref:`Object Libraries`.

Creating Relocatable Packages
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. |INTERFACE_PROPERTY_LINK| replace:: :prop_tgt:`INTERFACE_LINK_LIBRARIES`
.. include:: /include/INTERFACE_LINK_LIBRARIES_WARNING.txt
