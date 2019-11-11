# blog.xaviermaso.com

This is hosted on GitHub pages.


## Dev and pre visualization

Using [jekyll's docker image](https://github.com/envygeeks/jekyll-docker).

```bash
docker run --rm \
  --volume=$(pwd):/srv/jekyll \
  --net=host \
  -p 35729:35729 -p 4000:4000 \
  --env JEKYLL_ENV=development \
  -it jekyll/builder \
  jekyll serve --drafts --future --livereload
```

Then visit [localhost:4000](http://localhost:4000/).
