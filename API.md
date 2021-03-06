# API

* [Getting started](#getting-started)
* [`add`](#add)
* [`block.get`](#blockget)
* [`block.put`](#blockput)
* [`block.stat`](#blockstat)
* [`cat`](#cat)
* [`cp`](#cp) <sup>(MFS)</sup>
* [`dag.get`](#dagget)
* [`dag.put`](#dagput)
* [`dag.resolve`](#dagresolve)
* [`get`](#get)
* [`id`](#id)
* [`ls`](#ls) <sup>(MFS)</sup>
* [`mkdir`](#mkdir) <sup>(MFS)</sup>
* [`mv`](#mv) <sup>(MFS)</sup>
* [`read`](#read) <sup>(MFS)</sup>
* [`rm`](#rm) <sup>(MFS)</sup>
* [`start`](#start)
* [`stat`](#stat) <sup>(MFS)</sup>
* [`stop`](#stop)
* [`version`](#version)
* [`write`](#write) <sup>(MFS)</sup>

## Getting started

Create a new ipfsx node, that uses an `interface-ipfs-core` compatible `backend`.

### `ipfsx(backend, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| backend | `Ipfs`\|`IpfsApi` | Backing ipfs core interface to use |
| options | `Object` | (optional) options |
| options.ipldFormats | `Array` | IPLD formats for use with the DAG API. Default [ipld-raw](https://github.com/ipld/js-ipld-raw), [ipld-dag-pb](https://github.com/ipld/js-ipld-dag-pb), and [ipld-dag-cbor](https://github.com/ipld/js-ipld-dag-cbor). Note that setting this value with override the defaults. |

#### Returns

| Type | Description |
|------|-------------|
| `Promise<Node>` | An ipfsx node |

#### Example

```js
import ipfsx from 'ipfsx'
import IPFS from 'ipfs' // N.B. also works with ipfs-api!

const node = await ipfsx(new IPFS(/* options */))

// node.add(...)
// node.cat(...)
// etc...
```

## add

Add file data to IPFS.

### `node.add(input, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| input | `Buffer`\|`String`\|`Object<{content, path?}>`\|`Iterable`\|`Iterator` | Input files/data |
| options | `Object` | (optional) options |
| options.wrapWithDirectory | `Boolean` | Add the input files into a directory. This will be the last item output from the returned iterator. |

#### Returns

| Type | Description |
|------|-------------|
| `Iterator<{`<br/>&nbsp;&nbsp;`cid<`[`CID`](https://www.npmjs.com/package/cids)`>,`<br/>&nbsp;&nbsp;`path<String>`<br/>`}>` | Iterator of content IDs and paths of added files/data. It has an async `first()` and `last()` function for returning just the first/last item. |

#### Example

Add a string/buffer/object:

```js
for await (const res of node.add('hello world')) {
  console.log(res.cid)
}

// Or, since only one file is being added you can save some typing like this:
const { cid } = await node.add('hello world').first()
console.log(cid)
```

```js
const { cid } = await node.add(Buffer.from('hello world')).first()
console.log(cid)
```

```js
const { cid } = await node.add({ content: Buffer.from('hello world') }).first()
console.log(cid)
```

Add an (async) iterable/iterator:

```js
// Adding multiple files
// Note: fs.createReadStream is an ASYNC iterator!
const adder = node.add([
  { path: 'root/file1', content: fs.createReadStream(/*...*/) },
  { path: 'root/file2', content: fs.createReadStream(/*...*/) }
])

for await (const res of adder)
  console.log(res.cid, res.path)
```

```js
// Single file, regular iterator
const iterator = function * () {
  for (let i = 0; i < 10; i++)
    yield crypto.randomBytes()
}

const { cid } = await node.add(iterator()).first()
console.log(cid)
```

NOTE: if you have pull stream inputs, you can use [pull-stream-to-async-iterator](https://github.com/alanshaw/pull-stream-to-async-iterator) to convert them :D

## block.get

Fetch a raw block from the IPFS block store or the network via bitswap if not local.

### `node.block.get(cid)`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| cid |  `Buffer`\|`String`\|[`CID`](https://www.npmjs.com/package/cids) | CID of block to get |

#### Returns

| Type | Description |
|------|-------------|
| [`Block`](https://www.npmjs.com/package/ipfs-block) | Raw IPFS block |

#### Example

```js
const cid = new CID('zdpuAtpzCB7ma5zNyCN7eh1Vss1dHWuScf91DbE1ix9ZTbjAk')
const block = await node.block.get(cid)

Block.isBlock(block) // true
block.cid.equals(cid) // true
console.log(block.data) // buffer containing block data
```

## block.put

Put a block into the IPFS block store.

### `node.block.put(data, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| data | `Buffer`\|[`Block`](https://www.npmjs.com/package/ipfs-block)\|`Iterable`\|`Iterator` | Block data to store (multiple blocks if iterable/iterator) |
| options | `Object` | (optional) options (ignored if `data` is a `Block`) |
| options.format | `String` | [Multicodec name](https://github.com/multiformats/js-multicodec/blob/master/src/base-table.js) for the IPLD format that describes the data, default: 'raw' |
| options.cidVersion | `Number` | Version number of the CID to return, default: 1 |
| options.hashAlg | `String` | [Multihash hashing algorithm name](https://github.com/multiformats/js-multihash/blob/master/src/constants.js) to use, default: 'sha2-256' |
| options.hashLen | `Number` | Length to truncate the digest to |

#### Returns

| Type | Description |
|------|-------------|
| `Iterator<`[`Block`](https://www.npmjs.com/package/ipfs-block)`>` | Iterator that yields block objects. It has an async `first()` and `last()` function for returning just the first/last item. |

#### Example

Put single block from buffer of data:

```js
const data = Buffer.from('hello world')
const block = await node.block.put(data).first()

Block.isBlock(block) // true
block.cid.codec // raw
console.log(block.data.toString()) // hello world
```

Put an (async) iterable/iterator:

```js
const unixfs = require('js-unixfsv2-draft')
const block = await node.block.put(unixfs.dir(__dirname)).last()
// Last block is a unixfs2 directory
```

## block.stat

Get stats for a block.

### `node.block.stat(cid)`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| cid |  `Buffer`\|`String`\|[`CID`](https://www.npmjs.com/package/cids) | CID of block to get |

#### Returns

| Type | Description |
|------|-------------|
| `Promise<{size<Number>}>` | Block stats |

#### Example

```js
const block = await node.block.put(Buffer.from('hello world')).first()
const stats = await node.block.stat(block.cid)
console.log(block.cid.toBaseEncodedString(), stats)
// zb2rhj7crUKTQYRGCRATFaQ6YFLTde2YzdqbbhAASkL9uRDXn { size: 11 }
```

## cat

Get file contents.

### `node.cat(path, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String`\|`Buffer`\|[`CID`](https://www.npmjs.com/package/cids) | IPFS path or CID to cat data from |
| options | `Object` | (optional) options TODO |
| options.signal | [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal) | A signal that can be used to abort the request |

#### Returns

| Type | Description |
|------|-------------|
| `Iterator<Buffer>` | An iterator that can be used to consume all the data |

#### Example

```js
const { cid } = await node.add('hello world').first()

let data = Buffer.alloc(0)

for await (const chunk of node.cat(cid, options)) {
  data = Buffer.concat([data, chunk])
}

console.log(data.toString()) // hello world
```

Abort after a timeout:

```js
// Abort if not complete after 5 seconds
const controller = new AbortController()
setTimeout(() => controller.abort(), 5000)

// CID is valid, but (VERY probably) not on the network
const cid = 'zb2rhbry6PX6Qktru4SpxuVdkJm4mjdTqs8qzaPpvHk2tmx2w'
let data = Buffer.alloc(0)

try {
  for await (const chunk of node.cat(cid, { signal: controller.signal })) {
    data = Buffer.concat([data, chunk])
  }
} catch (err) {
  if (err.message === 'operation aborted') {
    console.log('Aborted after timeout')
  } else {
    throw err
  }
}
```

## cp

Copy files to MFS (Mutable File System).

### `node.cp(...from, to, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| from | `String` | IPFS path or MFS path to copy data from. |
| to | `String` | Destination MFS path to copy to. |
| options | `Object` | (optional) options |
| options.parents | `Boolean` | Automatically create parent directories if they don't exist, default: `false` |
| options.flush | `Boolean` | Immediately flush changes to disk, default: `true` |

**Note:**

If `from` has multiple values then `to` must be a directory.

If `from` has a single value and `to` exists and is a directory, `from` will be copied into `to`.

If `from` has a single value and `to` exists and is a file, `from` must be a file and the contents of `to` will be replaced with the contents of `from` otherwise an error will be returned.

If `from` is an IPFS path, and an MFS path exists with the same name, the IPFS path will be chosen.

#### Returns

| Type | Description |
|------|-------------|
| `Promise` | Resolves when the data has been copied |

#### Example

Copy MFS to MFS:

```js
// To copy a file
await node.cp('/src-file', '/dst-file')

// To copy a directory
await node.cp('/src-dir', '/dst-dir')

// To copy multiple files to a directory
await node.cp('/src-file1', '/src-file2', '/dst-dir')
```

Copy IPFS path to MFS:

```js
await node.cp('/ipfs/QmWGeRAEgtsHW3ec7U4qW2CyVy7eA2mFRVbk1nb24jFyks', '/hello-world.txt')
```

## dag.get

Retrieve data from an IPLD format node.

### `node.dag.get(path, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String`\|[`CID`](https://www.npmjs.com/package/cids)\|`Buffer` | IPFS path to the data that should be retrieved. Can also optionally be a CID instance or a Buffer containing an encoded CID |
| options | `Object` | (optional) options |
| options.signal | [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal) | A signal that can be used to abort the request |

#### Returns

| Type | Description |
|------|-------------|
| `Promise<?>` | The data in the node for the given path. |

#### Example

```js
const { cid } = await node.add([
  { content: 'hello world!', path: 'test/file1.txt' },
  { content: 'hello IPFS!', path: 'test/file2.txt' }
], { wrapWithDirectory: true }).last()

// node.add creates content with IPLD format dag-pb
// The dag-pb resolver returns DAGNode instances, see:
// https://github.com/ipld/js-ipld-dag-pb
const file1 = await node.dag.get(`/ipfs/${cid}/test/file1.txt`)
console.log(file1.data.toString()) // hello world!

const file2 = await node.dag.get(`/ipfs/${cid}/test/file2.txt`)
console.log(file2.data.toString()) // hello IPFS!
```

Traversing linked ndoes:

```js
const itemsCid = await node.dag.put({ apples: 5, pears: 2 }, { format: 'dag-cbor' }).first()
const basketCid = await node.dag.put({ items: itemsCid }, { format: 'dag-cbor' }).first()

const basket = await node.dag.get(`/ipfs/${basketCid}`)
console.log(basket) // { items: CID<zdpuAzD1F5ttkuXKHE9a2rcMCRdurmSzfmk9HFrz69dRm3YMd> }

const items = await node.dag.get(`/ipfs/${basketCid}/items`)
console.log(items) // { pears: 2, apples: 5 }

const apples = await node.dag.get(`/ipfs/${basketCid}/items/apples`)
console.log(apples) // 5
```

## dag.put

Store an IPLD format node.

### `node.dag.put(input, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| input | `?`\|`Iterable`\|`Iterator` | Data for DAG node to store (multiple nodes if iterable/iterator) |
| options | `Object` | (optional) options |
| options.format | `String` | [Multicodec name](https://github.com/multiformats/js-multicodec/blob/master/src/base-table.js) for the IPLD format that describes the data, default: 'raw' |
| options.hashAlg | `String` | [Multihash hashing algorithm name](https://github.com/multiformats/js-multihash/blob/master/src/constants.js) to use, default: 'sha2-256' |

#### Returns

| Type | Description |
|------|-------------|
| `Iterator<`[`CID`](https://www.npmjs.com/package/cids)`>` | Iterator that yields CID objects. It has an async `first()` and `last()` function for returning just the first/last item. |

#### Example

Store a `raw` DAG node:

```js
const cid = await node.dag.put(Buffer.from('Hello World!')).first()
console.log(cid.toString())
```

Store multiple DAG nodes from iterable (or async iterable):

```js
const iterable = [
  Buffer.from('Hello World!'),
  Buffer.from('Nice to meet ya!')
]

for await (const cid of node.dag.put(iterable)) {
  console.log(cid.toString())
}

// zb2rhfE3SX3q7Ha6UErfMqQReKsmLn73BvdDRagHDM6X1eRFN
// zb2rheUvNiPZtWauZfu2H1Kb64oRum1yuqkVLgMkD3j8nSCuX
```

Store with IPLD format `dag-cbor`:

```js
const cid = await node.dag.put({ msg: ['hello', 'world', '!'] }, { format: 'dag-cbor' }).first()
console.log(cid.toString()) // zdpuAqiXL8e6RZj5PoittgkYQPvE3Y4APxXD4sSdXYfS3x7P8

const msg0 = await node.dag.get(`/ipfs/${cid}/msg/0`)
console.log(msg0) // hello
```

Linking `dag-cbor` nodes:

```js
const itemsCid = await node.dag.put({ apples: 5, pears: 2 }, { format: 'dag-cbor' }).first()
const basketCid = await node.dag.put({ items: itemsCid }, { format: 'dag-cbor' }).first()

const basket = await node.dag.get(`/ipfs/${basketCid}`)
console.log(basket) // { items: CID<zdpuAzD1F5ttkuXKHE9a2rcMCRdurmSzfmk9HFrz69dRm3YMd> }

const items = await node.dag.get(`/ipfs/${basketCid}/items`)
console.log(items) // { pears: 2, apples: 5 }

const apples = await node.dag.get(`/ipfs/${basketCid}/items/apples`)
console.log(apples) // 5
```

## dag.resolve

Resolve an IPFS path to the CID of the DAG node that contains the data and the path within that node to the data (if any).

### `node.dag.resolve(path, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String`\|[`CID`](https://www.npmjs.com/package/cids)\|`Buffer` | IPFS path to resolve. Can also optionally be a CID instance or a Buffer containing an encoded CID |
| options | `Object` | (optional) options |
| options.signal | [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal) | A signal that can be used to abort the request |

#### Returns

| Type | Description |
|------|-------------|
| `Promise<{ cid<`[`CID`](https://www.npmjs.com/package/cids)`>, path<String>}>` | CID of the DAG node that contains the data and the path within that node to the data (if any). |

#### Example

```js
const itemsCid = await node.dag.put(
  { apples: { count: 5 }, pears: { count: 2 } },
  { format: 'dag-cbor' }
).first()
const basketCid = await node.dag.put({ items: itemsCid }, { format: 'dag-cbor' }).first()

const res = await node.dag.resolve(`/ipfs/${basketCid}/items/apples/count`)

console.log(res.cid.equals(itemsCid)) // true
console.log(`/ipfs/${res.cid}/${res.path}`)
// /ipfs/zdpuAqpAePJ59hkzjUZQQZysi9fPSGpPcSUUuz62TgCTLD5tb/apples/count
```

## get

Get file or directory contents.

#### Returns

| Type | Description |
|------|-------------|

### `node.get(path)`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String`\|`Buffer`\|[`CID`](https://www.npmjs.com/package/cids) | IPFS path or CID to get data from |
| options | `Object` | (optional) options |
| options.signal | [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal) | A signal that can be used to abort the request |

#### Returns

| Type | Description |
|------|-------------|
| `Iterator<{`<br/>&nbsp;&nbsp;`path<String>,`<br/>&nbsp;&nbsp;`content<Iterator<Buffer>>`<br/>`}>` | An iterator that can be used to consume all the data |

#### Example

Get a single file:

```js
const { cid } = await node.add('hello world').first()
const file = await node.get(cid).first()

let data = Buffer.alloc(0)

for await (const chunk of file.data) {
  data = Buffer.concat([data, chunk])
}

console.log(file.path, data)
```

Get a directory:

```js
const { cid } = await node.add([
  { path: 'root/LICENSE', content: fs.createReadStream('LICENSE') },
  { path: 'root/README.md', content: fs.createReadStream('README.md') }
]).last() // last item will be the root directory

for await (const file of node.get(cid)) {
  console.log('Path:', file.path)

  if (!file.content) continue // Directory has no content

  let data = Buffer.alloc(0)

  for await (const chunk of file.content) {
    data = Buffer.concat([data, chunk])
  }

  console.log('Data:', data)
}
```

## id

Get IPFS node identity information.

### `node.id()`

#### Returns

| Type | Description |
|------|-------------|
| `Promise<{`<br/>&nbsp;&nbsp;`id<String>,`<br/>&nbsp;&nbsp;`publicKey<String>,`<br/>&nbsp;&nbsp;`addresses<String[]>,`<br/>&nbsp;&nbsp;`agentVersion<String>,`<br/>&nbsp;&nbsp;`protocolVersion<String>`<br/>`}>` | Identity information |

#### Example

```js
const info = await node.id()
console.log(info)
/*
{ id: 'Qmak13He6dqmzWhJVoDGtgLBTZf9rrPXu2KrKk4RQBMKuD',
  publicKey:
   'CAASpgIwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCclFB7wWxstw3mBuXQV/FS3TjYK4BkQ2ChRPSuby33tj9s2gzC+A8Tu1fi6ZtKepFtIl879gtyV8Dtm3xnpDvBiMyvd+8ysKZ7c9dfFhTTedat2HcUPEq6uwxIHicuHuk8TXvzhZ6ovnkkn5b1OVEWicSsTKMBi4FoAySYFR2h15E0zDT5wK/EqENl6nzgd5gyEhuGRKh8jQly6/T8JABP1W4QJwJ7zaJWTRGDYrEK/T1eDcOF1vW0GjlNn0IbDsev8S5zzE2wXvtwZTLXzZlBEQ6S0wpxfWEHrQprMPEtZtv+I5LZDlZ1eecBNkiGEuWQAWBzr3XTJxkA2t0HLM5XAgMBAAE=',
  addresses:
   [ '/ip4/127.0.0.1/tcp/4002/ipfs/Qmak13He6dqmzWhJVoDGtgLBTZf9rrPXu2KrKk4RQBMKuD',
     '/ip4/127.0.0.1/tcp/4003/ws/ipfs/Qmak13He6dqmzWhJVoDGtgLBTZf9rrPXu2KrKk4RQBMKuD',
     '/ip4/192.168.0.17/tcp/4002/ipfs/Qmak13He6dqmzWhJVoDGtgLBTZf9rrPXu2KrKk4RQBMKuD' ],
  agentVersion: 'js-ipfs/0.31.7',
  protocolVersion: '9000' }
*/
```

## ls

List directory contents.

### `node.ls(path)`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String` | IPFS or MFS path of a _directory_ to list contents of. If `path` is an IPFS path, and an MFS path exists with the same name, the IPFS path will be chosen. |

#### Returns

| Type | Description |
|------|-------------|
| `Iterator<{`<br/>&nbsp;&nbsp;`cid<CID>,`<br/>&nbsp;&nbsp;`name<String>,`<br/>&nbsp;&nbsp;`size<Number>,`<br/>&nbsp;&nbsp;`type<String>`<br/>`}>` | An iterator that can be used to consume the listing<br/><br/>* `cid` is the content identifier for the file or directory<br/>* `name` is the name of the file or directory<br/>* `size` is the total size in bytes for the file/directory including all descendants and unixfs wrapper data. TODO: currently whatever js-ipfs or js-ipfs-api returns. Might be too small, too big or 0<br/>* `type` is either "directory" or "file"  |

#### Example

IPFS path listing:

```js
const res = await node.add([
  { path: `file1`, content: `${Math.random()}` },
  { path: `file2`, content: `${Math.random()}` },
  { path: `dir/file3`, content: `${Math.random()}` }
], { wrapWithDirectory: true }).last()

for await (const { cid, name, size, type } of node.ls(`/ipfs/${res.cid}`)) {
  console.log({ cid: cid.toString(), name, size, type })
}

/*
{ cid: 'QmdyDRruDdLc8ZFU8ekfsbgbL6QAVvfCXherJte1muRng6',
  name: 'dir',
  size: 78,
  type: 'directory' }
{ cid: 'QmSBZyCUSVv5ByZpDcC3MMd7fGFr1t7racVAGeo1P7wP4b',
  name: 'file1',
  size: 27,
  type: 'file' }
{ cid: 'QmTScjxfdvkoTre1ySsmmhXFUnikRGbevYPN97MFEzwLGP',
  name: 'file2',
  size: 27,
  type: 'file' }
*/
```

MFS path listing:

```js
const dirName = `/test-${Date.now()}`

await node.write(`${dirName}/file1`, `${Math.random()}`, { parents: true })
await node.write(`${dirName}/file2`, `${Math.random()}`, { parents: true })
await node.write(`${dirName}/dir/file3`, `${Math.random()}`, { parents: true })

for await (const { cid, name, size, type } of node.ls(dirName)) {
  console.log({ cid: cid.toString(), name, size, type })
}

/*
{ cid: 'Qmcao818uTXkZcV7JfwSKuBkTmGJUqkAY4Ur1mJDD2w87X',
  name: 'file2',
  size: 19,
  type: 'file' }
{ cid: 'QmZbdgi3ZesUbiHAGG5yaGqS7Dav3yUiGgiL4Lqee9DP3E',
  name: 'file1',
  size: 18,
  type: 'file' }
{ cid: 'QmXYZQedxLmp33HtkgNvSSHn7US3UhSNzYe8aF8N8wXpxH',
  name: 'dir',
  size: 0,
  type: 'directory' }
*/
```

## mkdir

Create an MFS (Mutable File System) directory.

### `node.mkdir(path, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String` | MFS path of the directory to create |
| options | `Object` | (optional) options |
| options.parents | `Boolean` | Automatically create parent directories if they don't exist, default: `false` |
| options.flush | `Boolean` | Immediately flush changes to disk, default: `true` |

#### Returns

| Type | Description |
|------|-------------|
| `Promise` | Resolved when the directory has been created |

#### Example

```js
await node.mkdir('/my-directory')
```

Automatically create parent directories if they don't exist:

```js
await node.mkdir('/path/to/my/directory', { parents: true })
```

## mv

Move MFS (Mutable File System) files and directories.

### `node.mv(...from, to, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String` | IPFS path or MFS path to move data from. |
| options | `Object` | (optional) options |
| options.parents | `Boolean` | Automatically create parent directories if they don't exist, default: `false` |
| options.flush | `Boolean` | Immediately flush changes to disk, default: `true` |

**Note:**

If `from` has multiple values then `to` must be a directory.

If `from` has a single value and `to` exists and is a directory, `from` will be moved into `to`.

If `from` has a single value and `to` exists and is a file, `from` must be a file and the contents of `to` will be replaced with the contents of `from` otherwise an error will be returned.

If `from` is an IPFS path, and an MFS path exists with the same name, the IPFS path will be chosen.

All values of `from` will be removed after the operation is complete, unless they are an IPFS path.

#### Returns

| Type | Description |
|------|-------------|
| `Promise` | Resolved when the files/directories have been moved |

#### Example

```js
await node.mkdir('/my-directory')
```

Automatically create parent directories if they don't exist:

```js
await node.mkdir('/path/to/my/directory', { parents: true })
```

## read

Read data from an MFS (Mutable File System) path.

### `node.read(path, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String` | MFS path of the _file_ to read (not a directory) |
| options | `Object` | (optional) options |
| options.offset | `Number` | Byte offset to begin reading at, default: `0` |
| options.length | `Number` | Number of bytes to read, default: `undefined` (read all bytes) |

#### Returns

| Type | Description |
|------|-------------|
| `Iterator<Buffer>` | An iterator that can be used to consume all the data |

#### Example

```js
let data = Buffer.alloc(0)
for (const chunk of node.read('/path/to/mfs/file')) {
  data = Buffer.concat([data, chunk])
}
```

## rm

Remove MFS (Mutable File System) files or directories.

### `node.rm(...path, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String` | MFS path to remove. |
| options | `Object` | (optional) options |
| options.recursive | `Boolean` | Remove directories recursively, default: `false` |

#### Returns

| Type | Description |
|------|-------------|
| `Promise` | Resolves when the files/directories have been removed |

#### Example

```js
// To remove a file
await node.rm('/src-file')

// To remove a directory
await node.rm('/src-dir', { recursive: true })
```

## start

Start the IPFS node.

### `node.start()`

#### Returns

| Type | Description |
|------|-------------|
| `Promise` | Resolved when the node has started |

#### Example

```js
const node = await ipfsx(new IPFS({ start: false }))
await node.start()
```

## stat

Get MFS (Mutable File System) file or directory status.

### `node.stat(path)`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String` | MFS path of the file or directory to get the status of |

#### Returns

| Type | Description |
|------|-------------|
| `Promise<{`<br/>&nbsp;&nbsp;`cid<CID>,`<br/>&nbsp;&nbsp;`size<Number>,`<br/>&nbsp;&nbsp;`cumulativeSize<Number>,`<br/>&nbsp;&nbsp;`blocks<Number>,`<br/>&nbsp;&nbsp;`type<String>`<br/>`}>` | Resolved when status has been retrieved.<br/><br/>* `cid` is the content identifier for the file or directory<br/>* `size` is the size in bytes of the data for the file/directory<br/>* `cumulativeSize` is the total size of the DAG node in bytes<br/>* `blocks` is the number of blocks this file/directory links to<br/>* `type` is "file" or "directory" |

#### Example

```js
const { cid, size, cumulativeSize, blocks, type } = await node.stat('/hello-world.txt')
```

## stop

Stop the IPFS node.

### `node.stop()`

#### Returns

| Type | Description |
|------|-------------|
| `Promise` | Resolved when the node has stopped |

#### Example

```js
const node = await ipfsx(new IPFS())
await node.stop()
```

## version

Get IPFS node version information.

### `node.version()`

#### Returns

| Type | Description |
|------|-------------|
| `Promise<{`<br/>&nbsp;&nbsp;`version<String>,`<br/>&nbsp;&nbsp;`repo<String>,`<br/>&nbsp;&nbsp;`commit<String>`<br/>`}>` | Version information |

#### Example

```js
const info = await node.version()
console.log(info)
// { version: '0.32.2', repo: 7, commit: '' }
```

## write

Write data to an MFS (Mutable File System) file.

### `node.write(path, input, [options])`

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| path | `String` | MFS path of the _file_ to write to |
| input | `Buffer`\|`String`\|`Iterable`\|`Iterator` | Input data |
| options | `Object` | (optional) options |
| options.offset | `Number` | Byte offset to begin writing at, default: `0` |
| options.length | `Number` | Number of bytes to write, default: `undefined` (write all bytes) |
| options.create | `Boolean` | Create file if not exists, default: `true` |
| options.parents | `Boolean` | Create parent directories if not exists, default: `false` |
| options.truncate | `Boolean` | Truncate the file after writing, default: `false` |
| options.rawLeaves | `Boolean` | Do not wrap leaf nodes in a protobuf, default: `false` |
| options.cidVersion | `Number` | CID version to use when creating the node(s), default: ~~1~~ FIXME: currently 0 due to https://github.com/ipfs/js-ipfs-mfs/issues/12 |

#### Returns

| Type | Description |
|------|-------------|
| `Promise` | Resolved when the write has been completed |

#### Example

Write a buffer/string:

```js
await node.write('/hello-world.txt', Buffer.from('hello world!'))
```

```js
await node.write('/example/hello-world.txt', 'hello world!', { parents: true })
```

Write a Node.js stream:

```js
await node.write('/example.js', fs.createReadStream(__filename), { create: true })
```
