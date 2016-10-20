# VVSFS (Very Very Simple File System)
Zhansong Li, Sina Eghbal
{u5844206, u5544352}@anu.edu.au
October 2016

## VVSFS
VVFSF is a very ver simple file system based on the simplistic RAM filesystem.
However, the filesystem does not support simple features such as removing directories,
truncating the files, showing the stat of the files, etc. A series of modifications
have been done on the VVSFS and certain features have been added to the filesystem which 
we will discuss in what follows.

### Features
- Removing and truncating files: The ability to remove and truncate files have been added to the filesystem.
This has been done by adding the missing unlink function to the directory operations.
```c
static int vvsfs_unlink (struct inode *dir, struct dentry *dentry);
```
The above function iterates through the directories and checks whether the directory's name matched
the name given to be deleted. Once it finds the matching name, it will break from the loop and
shifts the inodes after that one entry to the left.
- Truncating files: Truncating or shortening the files is done by setting their size in the
** setattr ** function. The setattr function is used to set/modify attributes of the inode. The attributes
are passed in a structure called  *iattr* and truncating will be done by calling truncate_setsize if the value of ATTR_SIZE is true.
Finally, we mark our inode to dirty so that our changes be recognized by the FS.

- Removing directories: Removing directories will be done by unlinking the inode and the dentry in the *vvsfs_rmdir*
function. The vvsfs_rmdir function is registered in the *dir_inode_operations* as rmdir.

- Creating directories: Creating directories is done by the *vvsfs_mkdir* directory which is merely calling the *vvsfs_mknod* function/
*vvsfs_mkdir* is also registered in vvsfs_dir_inode_operations as rmdir.

- Recording the file stats: To record the file stats we initially extended the very simple inode structure of vvsfs.
Therefore, we add the following entries to our structure.
```c
uid_t i_uid;
gid_t i_gid;
unsigned short mode;
```
Since we have changed the structure, we need to modify our MAXFILESIZE macro by subtracting the size of the variables we are using.
Hence, our file size will be calculated as follows:
```c
#define MAXFILESIZE (BLOCKSIZE - 3*sizeof (int) - sizeof (uid_t) - sizeof (gid_t) - sizeof (unsigned short))
// Block size - size of all the variables we're using in our structure.
```
This will provide us with the ability to store the user id, group id, and mode of our file.
Then we need to update the data while creating/writing into the file and read the data from vvsfs_inode.
In what follows, we will briefly discuss the steps required to store/retrieve the uid, gid, and mode data. 

1. Creating a new inode: When creating a new inode, we initialise its owner and group by calling the
```inode_init_owner (inode, dir, mode) ``` function. This way, we will be assigning the current user id to our inode.
However, since we are storing our inodes in out vvsfs_inode structure, we need to assign the values to the corresponding fields of the
aforementioned structure as following:
```c
struct inode *vvsfs_new_inode (const struct inode *dir, umode_t umode) { ...
  block.mode = (unsigned int)mode;
  block.i_uid = i_uid_read(inode);
  block.i_gid = i_gid_read(inode);
  block.size = sizeof (struct vvsfs_inode) - sizeof (char[MAXFILESIZE]) + sizeof (block.data);
  ...}
```
Hence once we store the inode in our blocks, we store the above information in our *vvsfs_inode*.
Also, we will need to update the inodes whenever we write into a file so we'll add the following lines the the following functions.
*vvsfs_mknod*
```c
 dirdata.size = (num_dirs + 1) * sizeof(struct vvsfs_dir_entry); 
 dirdata.mode = (unsigned int) mode;
```
*vvsfs_file_write*
```c
  inode->i_size = filedata.size;
  inode->i_mode = (umode_t)filedata.mode;
  inode->i_blocks = (filedata.size / BLOCKSIZE) + 1;
```
Finally, we fetch the values from our vvsfs_inode and copy them into the standard inode when we want to get the inode.
So we'll have:
```c
struct inode *vvsfs_iget(struct super_block *sb, unsigned long ino)
{...
	inode->i_size = filedata.size;
	// Write the uid and gid to the inode
	i_uid = filedata.i_uid;
	i_gid = filedata.i_gid;
	mode = (umode_t)filedata.mode;

	i_uid_write (inode, i_uid);
	i_gid_write (inode, i_gid);

	inode->i_ctime = inode->i_mtime = inode->i_atime = CURRENT_TIME;
	inode->i_mode = mode;
	...}
```
### Encryption
Encryption is done in the *encrypt_data* function. The function copies the information for our file system from
the location passed as its input, retrieves the password and applies a simple XOR function to encrypt/decrypt the data.
The return value of this function will be stores in one of the addresses _(to)_ passed to the function.


