#### case/commons-bcel/src/main/java/org/apache/bcel/classfile/LineNumber.java
```
    public LineNumber(final int start_pc, final int line_number) {
        this.start_pc = (short) start_pc;
        this.line_number = (short)line_number;
    }
```

This constructor is called by another constructor 
```
    LineNumber(final DataInput file) throws IOException {
        this(file.readUnsignedShort(), file.readUnsignedShort());
    }
```
`file.readUnsignedShort()` is unsigned short, therefore the input is in [0, 65535]

The first constructor is also called in  commons-bcel/src/main/java/org/apache/bcel/generic/LineNumberGen.java

```
    public LineNumber getLineNumber() {
        return new LineNumber(ih.getPosition(), src_line);
    }
```

`ih.getPosition()` is possible to be -1. Therefore, the input `start_pc` has a possible interval of [-1, 65535]

&nbsp;

&nbsp;

#### src/main/java/org/apache/bcel/classfile/Signature.java :

```
private static void matchIdent( final MyByteArrayInputStream in, final StringBuilder buf ) {    
    ...
    
    final StringBuilder buf2 = new StringBuilder();
    ch = in.read();
    do {
       buf2.append((char) ch);
       ch = in.read();
       //System.out.println("within ident:"+ (char)ch);
    } while ((ch != -1) && (Character.isJavaIdentifierPart((char) ch) || (ch == '/')));
    buf.append(buf2.toString().replace('/', '.'));
    //System.out.println("regular return ident:"+ (char)ch + ":" + buf2);
    if (ch != -1) {
       in.unread();
    }
}
```

The first iteration of the do-while loop directly convert the result of `in.read()` to `char`, which is unsafe.


&nbsp;

&nbsp;

#### src/main/java/org/apache/bcel/classfile/Signature.java :

```
private static void matchGJIdent( final MyByteArrayInputStream in, final StringBuilder buf ) {
    ...
    ch = in.read();
    if (identStart(ch)) {
        in.unread();
        matchGJIdent(in, buf);
    } else if (ch == ')') {
        in.unread();
        return;
    } else if (ch != ';') {
        throw new RuntimeException("Illegal signature: " + in.getData() + " read " + (char) ch);
    }
    ...
}
```

In the throw-statement, `ch` is converted to `char` without EOF test 

&nbsp;

&nbsp;


#### src/main/java/org/apache/bcel/classfile/Utility.java :
```
@Override
public int read( final char[] cbuf, final int off, final int len ) throws IOException {
   for (int i = 0; i < len; i++) {
       cbuf[off + i] = (char) read();
   }
   return len;
}
```
The return of `read()` is directly converted to `char`, while `read()` is possibly equal to -1, as its definition is

```
@Override
public int read() throws IOException {
   final int b = in.read();
   if (b != ESCAPE_CHAR) {
       return b;
   }
   final int i = in.read();
   if (i < 0) {
       return -1;
   }
   if (((i >= '0') && (i <= '9')) || ((i >= 'a') && (i <= 'f'))) { // Normal escape
       final int j = in.read();
       if (j < 0) {
           return -1;
       }
       final char[] tmp = {
               (char) i, (char) j
       };
       final int s = Integer.parseInt(new String(tmp), 16);
       return s;
   }
   return MAP_CHAR[i];
}
```

&nbsp;

&nbsp;


#### src/main/java/org/apache/bcel/classfile/Utility.java :
```
public static byte[] decode(final String s, final boolean uncompress) throws IOException {
   byte[] bytes;
   try (JavaReader jr = new JavaReader(new CharArrayReader(s.toCharArray()));
           ByteArrayOutputStream bos = new ByteArrayOutputStream()) {
       int ch;
       while ((ch = jr.read()) >= 0) {  // jr.read() returns [-1, 65535]
           bos.write(ch);   // input is required [0, 255]
       }
       bytes = bos.toByteArray();
   }
   ...
}
```

`jr.read()` returns a value in [-1, 65535]. After the comparison `(ch = jr.read()) >= 0`, in the then-branch, `ch` is refined to [0, 65535]. 
`bos.write()` takes an argument in [0, 255]. Passing `ch` as the argument violates the specification.


&nbsp;

&nbsp;

#### src/main/java/org/apache/bcel/generic/LOOKUPSWITCH.java :
```
    public LOOKUPSWITCH(final int[] match, final InstructionHandle[] targets, final InstructionHandle defaultTarget) {
        super(org.apache.bcel.Const.LOOKUPSWITCH, match, targets, defaultTarget);
        /* alignment remainder assumed 0 here, until dump time. */
        final short _length = (short) (9 + getMatch_length() * 8);
        super.setLength(_length);
        setFixed_length(_length);
    }
```
`getMatch_length()` is the length of the input array `match`. `(short) (9 + getMatch_length() * 8)` is not safe.


```
    @Override
    protected void initFromFile( final ByteSequence bytes, final boolean wide ) throws IOException {
        super.initFromFile(bytes, wide); // reads padding
        final int _match_length = bytes.readInt();
        setMatch_length(_match_length);
        final short _fixed_length = (short) (9 + _match_length * 8);
        setFixed_length(_fixed_length);
        final short _length = (short) (_match_length + super.getPadding());
        super.setLength(_length);
        super.setMatches(new int[_match_length]);
        super.setIndices(new int[_match_length]);
        super.setTargets(new InstructionHandle[_match_length]);
        for (int i = 0; i < _match_length; i++) {
            super.setMatch(i, bytes.readInt());
            super.setIndices(i, bytes.readInt());
        }
    }
```

` bytes.readInt()` may return arbitrary integer, which is assigned to `_match_length`. `(short) (9 + _match_length * 8)` and `(short) (_match_length + super.getPadding())` are not safe.


&nbsp;

&nbsp;


#### src/main/java/org/apache/bcel/generic/Select.java :
```
@Override
    protected int updatePosition( final int offset, final int max_offset ) {
        setPosition(getPosition() + offset); // Additional offset caused by preceding SWITCHs, GOTOs, etc.
        final short old_length = (short) super.getLength();
        /* Alignment on 4-byte-boundary, + 1, because of tag byte.*/
        padding = (4 - ((getPosition() + 1) % 4)) % 4;
        super.setLength((short) (fixed_length + padding)); // Update length
        return super.getLength() - old_length;
    }
```
` fixed_length` may be assigned arbitrary integer. `(short) (fixed_length + padding)` is not safe.


&nbsp;

&nbsp;


#### src/main/java/org/apache/bcel/generic/TABLESWITCH.java :
```
    public TABLESWITCH(final int[] match, final InstructionHandle[] targets, final InstructionHandle defaultTarget) {
        super(org.apache.bcel.Const.TABLESWITCH, match, targets, defaultTarget);
        /* Alignment remainder assumed 0 here, until dump time */
        final short _length = (short) (13 + getMatch_length() * 4);
        super.setLength(_length);
        setFixed_length(_length);
    }
```

`getMatch_length()` is the length of the input array `match`. `(short) (13 + getMatch_length() * 4)` is not safe.

```
@Override
    protected void initFromFile( final ByteSequence bytes, final boolean wide ) throws IOException {
        super.initFromFile(bytes, wide);
        final int low = bytes.readInt();
        final int high = bytes.readInt();
        final int _match_length = high - low + 1;
        setMatch_length(_match_length);
        final short _fixed_length = (short) (13 + _match_length * 4);
        setFixed_length(_fixed_length);
        super.setLength((short) (_fixed_length + super.getPadding()));
        super.setMatches(new int[_match_length]);
        super.setIndices(new int[_match_length]);
        super.setTargets(new InstructionHandle[_match_length]);
        for (int i = 0; i < _match_length; i++) {
            super.setMatch(i, low + i);
            super.setIndices(i, bytes.readInt());
        }
    }
```
`_match_length` may be arbitrary integer. `(short) (13 + _match_length * 4)` and `(short) (_fixed_length + super.getPadding())` are not safe.
