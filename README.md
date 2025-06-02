# Commands for monitoring manual build progress:
```bash
git ls-files --stage
git cat-file -p 7f118ff7695c4888a5ca943591eb013e40a63187
```

# Manual build:
## GIT INIT
```bash
mkdir -p .git/{objects,refs/heads}      # Create base folders - Git won't work without them
echo "ref: refs/heads/main" > .git/HEAD # Specify the current branch - without this, Git won't know where we are
```

## GIT ADD
```bash
echo -n "Hello, Git!" > hello.txt                                          # Create a file to add
content="Hello, Git!"                                                      # Save the contents to a variable
size=${#content} # Calculate the string size - needed for the header
sha1=$(printf "blob %d\0%s" "$size" "$content" | sha1sum | cut -d ' ' -f1) # Create an object and calculate the SHA-1 hash
mkdir -p ".git/objects/${sha1:0:2}" # Create a folder using the first 2 characters of the hash

printf "blob %d\0%s" "$size" "$content" | python3 -c "import sys, zlib; sys.stdout.buffer.write(zlib.compress(sys.stdin.buffer.read()))" > ".git/objects/${sha1:0:2}/${sha1:2}"                                       # Compress and save the object

git update-index --add --cacheinfo 100644 $sha1 hello.txt                  # Add to index - without this commit will not see the file

```

## GIT COMMIT
```bash
file_sha1="$sha1"                                                                         # Take the file hash from the previous step
printf "100644 hello.txt\0" > temp_tree                                                   # Write the rights and name
echo -n "$file_sha1" | xxd -r -p >> temp_tree # Add binary SHA
tree_size=$(wc -c < temp_tree) # Calculate the REAL file size
tree_sha1=$(printf "tree %d\0" "$tree_size" | cat - temp_tree | sha1sum | cut -d ' ' -f1) # Hash with the correct size
mkdir -p ".git/objects/${tree_sha1:0:2}" # Directory for tree
printf "tree %d\0" "$tree_size" | cat - temp_tree | python3 -c "import sys, zlib; sys.stdout.buffer.write(zlib.compress(sys.stdin.buffer.read()))" > ".git/objects/${tree_sha1:0:2}/${tree_sha1:2}" # Save
rm temp_tree # Remove temporary file

commit_content="tree $tree_sha1
author Vasya <v@test.com> 1700000000 +0000
committer Vasya <v@test.com> 1700000000 +0000

Initial commit"                                                                                         # Commit content

commit_sha1=$(printf "commit %d\0%s" "${#commit_content}" "$commit_content" | sha1sum | cut -d ' ' -f1) # Create and hash
mkdir -p ".git/objects/${commit_sha1:0:2}"                                                              # Directory for commit
printf "commit %d\0%s" "${#commit_content}" "$commit_content" | python3 -c "import sys, zlib; sys.stdout.buffer.write(zlib.compress(sys.stdin.buffer.read()))" > ".git/objects/${commit_sha1:0:2}/${commit_sha1:2}" # Save

echo "$commit_sha1" > .git/refs/heads/main                                                              # Update branch
```

## GIT LOG
```bash
commit_hash=$(cat .git/refs/heads/main) # Read last commit hash
python3 -c "import zlib; print(zlib.decompress(open('.git/objects/${commit_hash:0:2}/${commit_hash:2}', 'rb').read()).decode())" # Read and decompress commit object
```
