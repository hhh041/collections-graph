#HashMap Analysis

### key point
1. data struct
```
// entry array
HashMap{
Set keySet
Collection values
Entry[] table
int size
}

// linked list
Entry{
Entry next
}
```

2. hash
- string hash using Hashing.Holder.LANG_ACCESS.getStringHash32
- numberic hash using
  - h ^=(xor) k.hashCode()
  - h ^= (h >>> 20) ^ (h >>> 12);
  - return h ^ (h >>> 7) ^ (h >>> 4);

3. put
 - hash
 - get index by hash & (length-1)
 - iterator linked list while hash hit the target

4. resize
 - if ((size >= threshold) && (null != table[index]) then resize
 - threshold = capacity * loadFactor
 - transfer

5. transfer
 - outer iterator table
 - inner iterator linked list(Entry)
 -
 ```
                 Entry<K,V> next = e.next;
                 e.next = newTable[i];
                 newTable[i] = e;
                 e = next;
 ```
 - may cause endless loop

### code snippet
#### put
```
    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }

    void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }

    void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }
```
#### resize
```

    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }

    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```

#### hash
```
    final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
```

#### indexFor
```
    static int indexFor(int h, int length) {
        return h & (length-1);
    }
```

#### get
```
    public V get(Object key) {
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);

        return null == entry ? null : entry.getValue();
    }

    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }

        int hash = (key == null) ? 0 : hash(key);
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }
```
#### remove
```
    final Entry<K,V> removeEntryForKey(Object key) {
        if (size == 0) {
            return null;
        }
        int hash = (key == null) ? 0 : hash(key);
        int i = indexFor(hash, table.length);
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        while (e != null) {
            Entry<K,V> next = e.next;
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                if (prev == e)
                    table[i] = next;
                else
                    prev.next = next;
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }
```

### class diagram
@startuml

interface Iterator{
boolean hasNext()
E next()
void remove()
}

interface Map.Entry{
E getKey()
V getValue()
setValue(V)
equals()
hashCode()
}

abstract HashIterator{
Entry next
int expectedModCount
int index
Entry current

Entry nextEntry()
boolean hasNext()
void remove()
}

HashIterator "1" *-- "many" Entry : contains

Iterator <|.. HashIterator
HashIterator <|-- KeyInterator
class KeyInterator{
next()
}
HashIterator <|-- ValueInterator
class ValueInterator{
next()
}
HashIterator <|-- EntryInterator
class EntryInterator{
next()
}

abstract AbstractSet{
equals()
hashCode()
removeAll()
}

AbstractSet <|-- KeySet
class KeySet{
size()
contains()
remove()
clear()
}

AbstractSet <|-- EntrySet
class EntrySet{
size()
contains()
remove()
clear()
}

abstract AbstractCollection{
iterator()
size()
isEmpty()
contains()
add()
remove()
clear()
}

AbstractCollection <|-- Values

class Values{
size()
contains()
clear()
}


Map.Entry <|.. Entry
class Entry{
key
value
Entry next
hash

getKey()
getValue()
setValue()
equals()
hashCode()
}

class HashMap{
Set keySet
Collection values
Entry[] table
int size

get()
put()
resize()
transfer()
hash()
indexFor()
}


HashMap "1" o-- "many" Entry : aggregation


@enduml
