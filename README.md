# pizzapwn.ch

For of the al-folio template, see [al-folio](https://github.com/alshedivat/al-folio) for
the ctf website.

## Local Development

```bash
$ docker compose pull
$ docker compose up
```

Note that when you run it for the first time, it will download a docker image of size 400MB or so.

To see the template running, open your browser and go to `http://localhost:8080`. You can then modify the content of the
website and changes will be reflected immediately.

More Install Infos: Check out the [installation guide](INSTALL.md) for instructions on how to set up a local development environment.

## Creating new blog posts

New blog posts can be created by adding a new markdown file in the `_posts` directory.
The filename should follow the format `YYYY-MM-DD-title.md`, where `YYYY-MM-DD` is the date of the post and `title`
is a short, descriptive title for the post.

Images for the post can be added to the `assets` directory, and can be referenced in the markdown file using
the path `assets/posts/image.png`.

**Prettier**
Please run the linter/prettier before submitting your code.

```
npx prettier --check .
# Auto-Fix the prettiere issues with this command:
npx prettier --write .
```


## Contributing

Contributions are very welcome! If you want to submit your changes, please
fork the repository, make your changes, and submit a pull request from your fork.

## License

The theme is available as open source under the terms of the [MIT License](https://github.com/alshedivat/al-folio/blob/main/LICENSE).
Originally, **al-folio** was based on the [\*folio theme](https://github.com/bogoli/-folio) (published by [Lia Bogoev](https://liabogoev.com) and under the MIT license). Since then, it got a full re-write of the styles and many additional cool features.
