# Blog backend stuff

## Install hugo

```bash
sudo apt install hugo
```

## Hugo Theme

Go to directory `ashtrace_blog/`, create directory `themes` and run `git clone https://gitlab.com/gabmus/hugo-ficurinia.git`

## Create a post

Run

```bash
hugo new --kind page posts/<folder_name>/index.md
# eg: hugo new --kind page posts/tracing_file_descriptors/index.md
```

## View it localay

Run

```bash
hugo server
```

## Create and export static page to blog

```bash
hugo -t hugo-ficurinia
```

## Publish

Go to public directory and push the changes

```bash
cd public
git add .
git commit -m "<message here>"
git push origin main
```
