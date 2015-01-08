# Code one can put into a .gdbinit file to make gdb work better with deal.II

This is code that makes gdb aware of some of the deal.II classes and able to print their elements in a debugger in a way that is more easily readable.
See the FrequentlyAskedQuestions for more information.

```
set print pretty 1

python

import gdb
import itertools
import re

# Try to use the new-style pretty-printing if available.
_use_gdb_pp = True
try:
    import gdb.printing
except ImportError:
    _use_gdb_pp = False

class PointPrinter:
    "Print dealii::Point"

    def __init__ (self, typename, val):
        self.typename = typename
        self.val = val

    def to_string (self):
        return '%s' % self.val[class TensorPrinter:
    "Print dealii::Tensor"

    def __init__ (self, typename, val):
        self.typename = typename
        self.val = val

    def to_string (self):
        1. we could distinguish between Tensor<1,dim> and Tensor<2,dim>
        1. by asking self.val.type.template_argument(0) but unfortunately
        1. template_argument does not handle value arguments...
        try:
	  return self.val['values']('values'])
	except:
	  return self.val[class TriaIteratorPrinter:
    "Print dealii::TriaIterator"

    def __init__ (self, typename, val):
        self.typename = typename
        self.val = val

    def to_string (self):
        if re.compile('.*DoF.*').match('%s' % self.val.type.template_argument(0)):
          return ('{\n  triangulation = %s,\n  dof_handler = %s,\n  level = %d,\n  index = %d\n}' %
	          (self.val['accessor']('subtensor'])[self.val['accessor']('tria'],)[self.val['accessor']('dof_handler'],)[self.val['accessor']('present_level'],)[else:
          return ('{\n  triangulation = %s,\n  level = %d,\n  index = %d\n}' %
	          (self.val['accessor']('present_index'])))[self.val['accessor']('tria'],)[self.val['accessor']('present_level'],)[class VectorPrinter:
    "Print dealii::Vector"

    class _iterator:
        def __init__ (self, start, size):
            self.start = start
            self.size = size
            self.count = 0

        def __iter__(self):
            return self

        def next(self):
            count = self.count
            self.count = self.count + 1
	    self.start = self.start + 1
            if self.count == self.size:
                raise StopIteration
            elt = self.start.dereference()
            return ('[%d]('present_index'])))' % count, elt)

    def __init__ (self, typename, val):
        self.typename = typename
        self.val = val

    def children(self):
        return self._iterator(self.val[self.val['vec_size']('val'],))

    def to_string_x (self):
        return ('%s[= {%s}' % (self.val.type.template_argument(0),
                                   self.val['vec_size'](%d]),
				   self.val[def to_string (self):
        return ('%s[%d]('val'].dereference())))' % (self.val.type.template_argument(0),
                            self.val[def display_hint(self):
        return 'array'



# A "regular expression" printer which conforms to the
# "SubPrettyPrinter" protocol from gdb.printing.
class RxPrinter(object):
    def __init__(self, name, function):
        super(RxPrinter, self).__init__()
        self.name = name
        self.function = function
        self.enabled = True

    def invoke(self, value):
        if not self.enabled:
            return None
        return self.function(self.name, value)


# A pretty-printer that conforms to the "PrettyPrinter" protocol from
# gdb.printing.  It can also be used directly as an old-style printer.
class Printer(object):
    def __init__(self, name):
        super(Printer, self).__init__()
        self.name = name
        self.subprinters = []('vec_size'])))
        self.lookup = {}
        self.enabled = True
        self.compiled_rx = re.compile('^([def add(self, name, function):
        1. A small sanity check.
        1. FIXME
        if not self.compiled_rx.match(name + '<>'):
            raise ValueError, 'libstdc++ programming error: "%s" does not match' % name
        printer = RxPrinter(name, function)
        self.subprinters.append(printer)
        self.lookup[name](a-zA-Z0-9_:]+)<.*>$')) = printer

    @staticmethod
    def get_basic_type(type):
        1. If it points to a reference, get the reference.
        if type.code == gdb.TYPE_CODE_REF:
            type = type.target ()

        1. Get the unqualified type, stripped of typedefs.
        type = type.unqualified ().strip_typedefs ()

        return type.tag

    def __call__(self, val):
        typename = self.get_basic_type(val.type)
        if not typename:
            return None

        1. All the types we match are template types, so we can use a
        1. dictionary.
        match = self.compiled_rx.match(typename)
        if not match:
            return None

        basename = match.group(1)
        if basename in self.lookup:
            return self.lookup[basename].invoke(val)

        1. Cannot find a pretty printer.  Return None.
        return None

dealii_printer = None

def build_dealii_dictionary ():
    global dealii_printer

    dealii_printer = Printer("deal.II")
    dealii_printer.add ('dealii::Point', PointPrinter)
    dealii_printer.add ('dealii::Tensor', TensorPrinter)
    dealii_printer.add ('dealii::TriaRawIterator', TriaIteratorPrinter)
    dealii_printer.add ('dealii::TriaIterator', TriaIteratorPrinter)
    dealii_printer.add ('dealii::TriaActiveIterator', TriaIteratorPrinter)
    dealii_printer.add ('dealii::Vector', VectorPrinter)

def register_dealii_printers (obj):
    "Register deal.II pretty-printers with objfile Obj."

    global _use_gdb_pp
    global dealii_printer

    build_dealii_dictionary ()
    if _use_gdb_pp:
        gdb.printing.register_pretty_printer(obj, dealii_printer)
    else:
        if obj is None:
            obj = gdb
        obj.pretty_printers.append(dealii_printer)


register_dealii_printers (None)

end
```