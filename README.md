https://jimvella.github.io/techpdf/

## How to use

Add 3 squares to the pdf in Bluebeam (the lower left corner is the reference point) with a text value of "#techpdf <lat>, <long>"

Add togglable shading 200m and 300m around the Bowser shape annotation.

```
Bowser
#r1 200 rgb(255,0,0,0.2)
#r2 300 rgb(0,255,0,0.2)
```

The coordiates of all the other annoations in the document can then be interpolated using bayesian weights. Note that the geometry of the earth isn't considered, so the estimation gets worse as the distances get larger.

## Developing

```bash
npm run dev

# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building and Deployment

```bash
npm run build
```

Deployment is via Github pages. svelte.config.js has been configured to output the built static site to /docs for Github pages. Publishing is a matter of committing the changes to /docs and pushing to Github.
