# Imiron's public website

- You can use the nix flake to get all build dependencies.
- The website is built using [Hugo](https://gohugo.io/).
- The CI will automatically deploy the website to GitHub pages.
- We have made a custom theme (`themes/imi`).
- We use the [Bulma](https://bulma.io/) CSS framework.

## Development

- You can build the website with `hugo` commmand.
- If you want to include `draft` posts, use `hugo -D`.
- You can start the hugo development server with `hogo server`. This will launch a local server which will automatically update when you edit files.
- The CSS can be built with the command `npm run build-bulma` from within the `imi` theme folder (`themes/imi`).
- You can also run `npm run start` to automatically build the CSS file whenever `main.scss` is edited.
