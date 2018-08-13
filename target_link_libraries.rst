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

:ref:`Object Libraries` を、 ``target_link_libraries`` の（最初の）
``<target>`` 引数にして、そのソースが他のライブラリに依存していることを示すために
使うことができます。例えば、以下のコード

.. code-block:: cmake

  add_library(A SHARED a.c)
  target_compile_definitions(A PUBLIC A)

  add_library(obj OBJECT obj.c)
  target_compile_definitions(obj PUBLIC OBJ)
  target_link_libraries(obj PUBLIC A)

は、``obj.c`` を ``-DA -DOBJ`` 付きでコンパイルし、 ``obj`` の使用要件を確立します。
それは、それに依存しているものに伝搬します。

通常のライブラリと実行可能ファイルは、 :ref:`Object Libraries` にリンクして、
そのオブジェクトと使用要件を得ることができます。前記の例を続けると、以下のコード

.. code-block:: cmake

  add_library(B SHARED b.c)
  target_link_libraries(B PUBLIC obj)

は、``b.c`` を ``-DA -DOBJ`` 付きでコンパイルし、``b.c`` と ``obj.c`` から
得られるオブジェクトファイルで共用ライブラリ ``B`` を作ります。
そして、``B`` を ``A`` にリンクします。
さらに以下のコード

.. code-block:: cmake

  add_executable(main main.c)
  target_link_libraries(main B)

は、``main.c`` を ``-DA -DOBJ`` 付きでコンパイルし、実行可能ファイル ``main`` を
``B`` と ``A`` にリンクします。
オブジェクトライブラリの使用要件は、 ``B`` を通して推移的に伝搬しますが、
そのオブジェクトファイルは伝搬しません。

:ref:`Object Libraries` を他のライブラリに、「リンク」して、使用要件を
得ることはできますが、それはリンクステップを持たないので、そのオブジェクトファイルに対しては、
何も行われません。前記の例を続けると、以下のコード

.. code-block:: cmake

  add_library(obj2 OBJECT obj2.c)
  target_link_libraries(obj2 PUBLIC obj)

  add_executable(main2 main2.c)
  target_link_libraries(main2 obj2)

は、``obj2.c`` を ``-DA -DOBJ`` 付きでコンパイルして、
``main2.c`` と ``obj2.c`` から得られるオブジェクトファイルで ``main2`` 実行可能ファイル
を作り、 ``main2`` を ``A`` にリンクします。

言葉を変えて言えば、 :ref:`Object Libraries` がターゲットの
:prop_tgt:`INTERFACE_LINK_LIBRARIES` 属性に現れる時、それは、
:ref:`Interface Libraries` として扱われます。しかし、それがターゲットの
:prop_tgt:`LINK_LIBRARIES` 属性に現れる時、そのオブジェクトファイルも
リンクに含まれます。

Cyclic Dependencies of Static Libraries
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ライブラリの依存関係グラフは、通常は循環なし（DAG）です。しかし、相互に依存する
``STATIC`` ライブラリの場合、CMake はグラフが循環（強く結合したコンポーネント）を持つことを許します。
他のターゲットがそのライブラリの一つにリンクする場合、CMake は結合したコンポーネント全体を
繰り返します。例えば、以下のコードは、

.. code-block:: cmake

  add_library(A STATIC a.c)
  add_library(B STATIC b.c)
  target_link_libraries(A B)
  target_link_libraries(B A)
  add_executable(main main.c)
  target_link_libraries(main A)

``main`` を ``A B A B`` にリンクします。普通は一度の繰り返しで十分ですが、
悲劇的なオブジェクトファイルとシンボルの配置はそれ以上を要求することがあります。
そのような場合は、 :prop_tgt:`LINK_INTERFACE_MULTIPLICITY` ターゲット属性を使うか、
最後の ``target_link_libraries`` 呼び出しのコンポーネントを手作業で繰り返すことで
対処することができます。
しかし、もし二つのアーカイブが本当にそのように相互依存しているならば、それらは、
:ref:`Object Libraries` を使ってもよいですが、一つのアーカイブに結合するべきです。

Creating Relocatable Packages
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. |INTERFACE_PROPERTY_LINK| replace:: :prop_tgt:`INTERFACE_LINK_LIBRARIES`
.. include:: /include/INTERFACE_LINK_LIBRARIES_WARNING.txt
