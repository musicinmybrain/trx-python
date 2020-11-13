# tractography_file_format
Official repository to present specifications, showcase examples, discuss issues and keep track of everything.
Currently host only two file format propositions, the TRX file format (as memmap vs zarr). Anyone is free to contribute.

Try it out with:
```bash
pip install -e .
```


# Generals
- (Un)-Compressed Zip File or simple folder architecture
-- File architecture describe the data
-- Each file basename is the metadata’s name
-- Each file extension is the metadata’s dtype
-- Each file dimension is in the value between basename and metdata,  1-dimension array do not have to follow this convention for readability
- All arrays have a C-style memory layout (row-major)
--  Simplest representation, pure binary

# Header
Only (or mostly) for use-readability, read-time checks and broader compatibility.

- Dictionary in YAML
-- VOXEL_TO_RASMM
-- DIMENSIONS
-- NB_STREAMLINES
-- NB_POINTS

# Arrays
##### positions.float16
- Written in world space (RASMM)
-- Like TCK file 
- Should always be a float16/32/64
-- Default could be float16
- As contiguous 3D array (NBR_POINTS, 3)

##### offsets.uint64 
- Always uint64
- Two ways of knowing how many points there are:
-- Check the header
-- Positions array size / dtypes / 3

- To get streamlines lengths: append the total number of points to the end of offsets and to the differences between consecutive elements of the array (ediff1d in numpy). 

##### dpg (data_per_group)
- Each folder is the name of a group
- Not all metadata have to be present in all groups
- Always of size (1,) or (N,)

##### dpp (data_per_point)
- Always of size (NBR_POINTS, 1) or (NBR_POINTS, N)

##### dps (data_per_streamline)
- Always of size (NBR_STREAMLINES, 1) or (NBR_STREAMLINES, N)

##### Groups
- Groups are tables of indices that allow sparse & overlapping representation (clusters, connectomics, bundles)
- Recommended uint32 (as long as streamlines count is below 4 294 967 295)
- Allow to get a predefined streamlines subset from the memmaps efficiently
- Variables in sizes

### Example structure
complete_big_v4.trx
         ├── dpg
         │         ├── AF_L
         │         │         ├── mean_fa.float16
         │         │         ├── shuffle_colors.3.uint8
         │         │         └── volume.uint32
         │         ├── AF_R
         │         │         ├── mean_fa.float16
         │         │         ├── shuffle_colors.3.uint8
         │         │         └── volume.uint32
         │         ├── CC
         │         │         ├── mean_fa.float16
         │         │         ├── shuffle_colors.3.uint8
         │         │         └── volume.uint32
         │         ├── CST_L
         │         │         └── shuffle_colors.3.uint8
         │         ├── CST_R
         │         │         └── shuffle_colors.3.uint8
         │         ├── SLF_L
         │         │         ├── mean_fa.float16
         │         │         ├── shuffle_colors.3.uint8
         │         │         └── volume.uint32
         │         └── SLF_R
         │             ├── mean_fa.float16
         │             ├── shuffle_colors.3.uint8
         │             └── volume.uint32
         ├── dpp
         │         ├── color_x.uint8
         │         ├── color_y.uint8
         │         ├── color_z.uint8
         │         └── fa.float16
         ├── dps
         │         ├── algo.uint8
         │         ├── clusters_QB.uint16
         │         ├── commit_colors.3.uint8
         │         └── commit_weights.float32
         ├── groups
         │         ├── AF_L.uint32
         │         ├── AF_R.uint32
         │         ├── CC.uint32
         │         ├── CST_L.uint32
         │         ├── CST_R.uint32
         │         ├── SLF_L.uint32
         │         └── SLF_R.uint32
         ├── header.yaml
         ├── offsets.uint64
         └── positions.3.float16

```python
import numpy as np  
from trx_file_memmap import load, save, TrxFile

trx = load('complete_big_v4.trx')

# Access the header (dict) / streamlines (ArraySequences)
trx.header
trx.streamlines

# Access the dpp (dict) / dps (dict)
trx.data_per_point
trx.data_per_streamlines

# Access the groups (dict) / dpg (dict)
trx.groups
trx.data_per_group

# Get a random subset of 10000 streamlines
indices = np.arange(len(trx.streamlines._lengths))
np.random.shuffle(indices)
sub_trx = trx.select(indices[0:10000])
save(sub_trx, 'random_1000.trx')

# Get sub-groups only, from the random subset
for key in sub_trx.groups.keys():
	group_trx = sub_trx.get_group(key) 
    save(group_trx, '{}.trx'.format(key)) 

# Pre-allocate memmaps and append 100x the random subset
alloc_trx = TrxFile(nb_streamlines=1500000, nb_points=500000000, init_as=trx)
for i in range(100):
    alloc_trx.append(sub_trx)

# Resize to remove the unused portion of the memmap
alloc_trx.resize()
```
