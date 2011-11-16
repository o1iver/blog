public: yes
tags: [python, programming, love, meta]

Python: Runtime type (class) changes
====================================

*Not much need to explain here, just check out the example (I am changing the class of the object at runtime).*

.. sourcecode:: python

    class Female(object):
        def __init__(self, name):
            self.name = name

        def intro(self):
            print("Hi, I am %s and I am a woman!" % self.name)

    class Male(object):
        def __init__(self, name):
             self.name = name
     
         def intro(self):                                                                                              
             print("Hi, I am %s and I am a man!" % self.name)
                                                       
Let's try it!

.. sourcecode:: python

    >> me = Male("Oliver Stollmann")
    >> me.intro()
    Hi, I am Oliver Stollmann and I am a man!
    >> me.__class__ = Female
    >> me.intro()
    Hi, I am Oliver Stollmann and I am a woman!

