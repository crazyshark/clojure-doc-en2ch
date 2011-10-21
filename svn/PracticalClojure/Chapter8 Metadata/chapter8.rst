==============
Ԫ����(metadata)
==============


�ڴ����������������
=========================

����Ա����̸��metadata,�����ݵ����ݡ�metadata�ڲ�ͬ�������Ļ����ж����ǲ�һ���ġ�clojure�ṩ����metadata
    
����������ֵͬ�Ͳ�ͬԪ����(metadata)�Ķ�����Ϊ����ȵģ�������ͬ��hashֵ�������ǣ�Ԫ����(metadata)������clojure 
�����ݽṹһ�������в������������(immutable semantics)�����ı�����Ԫ����(metadata)������һ���µĶ��������ɵ�
������ԭʼ�Ķ�������ͬ��ֵ����ͬ��hashֵ����
 
���ı�һ��ֵʱ����Щ������ı�metadata,��Щ���ᡣ���»��ص����ۡ�

��дmetadata(Reading and Writing Metadata)
==================================================
 
Ĭ�ϻ����У�REPL�ǲ����ӡ��metadata�ġ���Ҫ��*\*print-meta\**���ó�true��Ϊ���ʾ��:: 
	
	(set! \*print-meta\* true)

�������*with-meta*������clojure��ķ���(symbol)���ڽ������ݽṹ����Ԫ���ݣ�metadata��,��meta������ȡԪ����(metadata)��::
     
    (with-meta obj meta-map)
    (meta obj)

with-meta�����������¶��󣬶����ֵ��*obj*,Ԫ����(metadata)��*meta-map*,*meta* �������ض���obj��Ԫ���ݵ�ͼ(��֪��ô�����metadata map)��
��������Ĵ��� :: 
    
    user=> (with-meta [1 2] {:about "A vector"})
    #^{:about "A vector"} [1 2]

��Ҳ������*vary-meta*�����Ԫ���ݵ�ͼ(metadata map):: 
    
    (vary-meta obj function & args)

vary-meta ����һ��������������ĺ�����Ӧ�õ�����ǰ��Ԫ���ݵ�ͼ(metadata map)�����᷵��һ������Ԫ���ݣ�metadata�����¶���
���磺����Ĵ��� ::

    user=> (def x (with-meta [3 4] {:help "Small vector"}))
    user=> x
    #^{:help "Small vector"} [3 4]
    user=> (vary-meta x assoc :help "Tiny vector")
    #^{:help "Tiny vector"} [3 4]

ע������������ֵͬ��ͬԪ���ݵĶ������ȵġ���������clojure���"="�������ԣ����ǣ����ڴ��в���ͬһ���󣨿���clojure���identical?�������ԣ�::

    user=> (def v [1 2 3])
    user=> (= v (with-meta v {:x 1}))
    true
    user=> (identical? v (with-meta v {:x 1}))
    false

��ס����ֻ����clojure���һЩ�����������Ԫ����(metadata),�����б�(list),����(vector),����(symbol) (functions in Clojure 1.2)��
java�࣬��String��Number,�ǲ�֧�ֵġ�  
    


metadata�������(Metadata-Preserving Operations)
==================================================
һЩ"�ı䡱��������ݽṹʱ��洢����Ԫ���ݣ�metadata������Щ�򲻻ᡣ
���磬*conj*���������б�ʱ��洢��Ԫ���ݣ�metadata����������*cons*�����򲻻� ::

    user=> (def x (with-meta (list 1 2) {:m 1}))
    user=> x
    #^{:m 1} (1 2)
    user=> (conj x 3)
    #^{:m 1} (3 1 2)
    user=> (cons 3 x)
    (3 1 2) ;; no metadata!




(Read-Time Metadata)
==================================================
    
    user=> #^{:m 1} [1 2]
    #^{:m 1} [1 2]



    user=> #^{:m 1} (list 1 2)
    (1 2) ;; no metadata!


�������metadata
==================================================

    
    user=> (meta (var or))
    {:ns #<Namespace clojure.core>
    :name or
    :file "clojure/core.clj"
    :line 504
    :doc "Evaluates exprs one at a time..."
     ([] [x] [x & next])
    :macro true}


    user=> (def #^{:doc "My cool thing"} \*thing\*)
    #'user/\*thing\*
    user=> (:doc (meta (var \*thing\*)))
    "My cool thing"



    => (doc \*thing\*)
    -------------------------
    user/\*thing\*
    nil
        My cool thing


    (defn name doc-string meta-map [params] ...)
    (defmacro name doc-string meta-map [params] ...)


    (def #^{:private true} \*my-private-var\*) ;; for Vars
    (defmacro my-private-macro {:private true} [args] ...)
    ;; for macros



�����������metadata
==================================================

    (alter-meta! iref f & args)


    (alter-meta! (var for) assoc :note "Not a loop!")
    {:note "Not a loop!" :macro true, ...
    user=> (:note (meta (var for)))
    "Not a loop!"

    user=> (def r (ref nil :meta {:about "This is my ref"}))
    user=> (meta r)
    {:about "This is my ref"}



�ܽ�



===================================================

Ԫ����(metadata)�������ֲ�ͬ�����ԡ�
