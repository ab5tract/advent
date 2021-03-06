# RFC 54, by Damian Conway: Operators: Polymorphic comparisons

This RFC was originally proposed on August 7th 2020 and frozen in six weeks.

It described a frustration with comparison operations in perl. The expression:

	"cat" == "dog"   #True

Perl (and now Raku) has excellent support for generic programming because of dynamic typing, generic data types and interface polymorphism. Just about the only place where that DWIM genericity breaks down is in comparisons. There is no generic way to specify an ordering on dynamically typed data. For example, one cannot simply code a generic BST insertion method. Using <=> for the comparison will work well for numeric keys, but fails miserably on most string-based keys because <=> will generally return 0 for most pairs of strings. The above code would work correctly however if <=> detected the string/string comparison and automagically used cmp instead.

# The Raku implementation is as follows:

There are three built-in comparison operators that can be used for sorting. They are sometimes called three-way comparators because they compare their operands and return a value meaning that the first operand should be considered less than, equal to or more than the second operand for the purpose of determining in which order these operands should be sorted. The leg operator coerces its arguments to strings and performs a lexicographic comparison. The <=> operator coerces its arguments to numbers (Real) and does a numeric comparison. The aforementioned cmp operator is the “smart” three-way comparator, which compares strings with string semantics and numbers with number semantics.[1]

The allomorph types IntStr, NumStr, RatStr and ComplexStr may be created as a result of parsing a string quoted with angle brackets... 

	my $f = <42.1>; say $f.^name; # OUTPUT: «RatStr␤»

Returns either Order::Less, Order::Same or Order::More object. Compares Pair objects first by key and then by value etc.

Evaluates Lists by comparing element @a[$i] with @b[$i] (for some Int $i, beginning at 0) and returning Order::Less, Order::Same, or Order::More depending on if and how the values differ. If the operation evaluates to Order::Same, @a[$i + 1] is compared with @b[$i + 1]. This is repeated until one is greater than the other or all elements are exhausted. If the Lists are of different lengths, at most only $n comparisons will be made (where $n = @a.elems min @b.elems). If all of those comparisons evaluate to Order::Same, the final value is selected based upon which List is longer.

If $a eqv $b, then $a cmp $b always returns Order::Same. Keep in mind that certain constructs, such as Sets, Bags, and Mixes care about object identity, and so will not accept an allomorph as equivalent of its components alone.

# Now, we can leverage the Raku sort syntax to suit our needs:

	my @sorted = sort { $^a cmp $^b }, @values;
	my @sorted = sort -> $a, $b { $a cmp $b }, @values;
	my @sorted = sort * cmp *, @values;
	my @sorted = sort &infix:cmp», @values;

And neatly avoid duplication in a functional style:

	# sort case-insensitively
	say sort { $^a.lc cmp $^b.lc }, @words;
	#          ^^^^^^     ^^^^^^  code duplication
	# sort case-insensitively
	say sort { .lc }, @words;
	
So this solution to RFC54 smoothly combines many of the individual capabilities of Raku - classes, allomorphs, dynamic typing, interface polymorphism, and functional programming to produce a set of practical solutions to suit your coding style.


[1] This is ass-backwards from the RFC54 request with 'cmp' as the dynamic "apex", degenerating to '<=>' for Numeric or 'leg' for Lexicographic variants.  
