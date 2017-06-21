# Developers.strava.com (β)

This code contains the sources needed to generate the content of [developers.strava.com]
(https://developers.strava.com).

Specifically, this repository implements a [Hugo](https://gohugo.io) static website, whose content
is pre-generated from a set of [Swagger](swagger.io) definitions that describe the Strava API.
Swagger is an open-source set of tools that aim to facilitate the management of REST APIs by:

- Defining a specification for describing API endpoints (URL, method, parameters, etc…)
- Implementing a subset of the JSON-Schema specification that enables describing the models (i.e.
  the payloads) that the API consumes and returns.

This model enables anyone to generate up-to-date, working client libraries for the Strava API and
matching documentation. As such, this repository serves two purposes:

- Be the source of truth for endpoints and models exposed by the Strava API.
- Be a place of exchange for external developers to continuously provide feedback, track bugs and
feature requests and (optionally) contribute back.

From these, we intend to provide developers with a better support and tighter feedback loop with
the Strava engineering team, as well as a place of exchange for the community of developers using
the Strava API.

## Compiling and running the site

We rely on [Docker](https://www.docker.com/) to build and run the site. Download and install Docker
for your machine and then build and run the image:

    $ docker build --rm -t strava/developers.strava.com .
    $ docker run -p 1313:1313 strava/developers.strava.com:latest

## Generation of code and documentation

### Client code

To generate client code, you first need to install Swagger. On macOS, you may use Homebrew:

    $ brew install swagger-codegen maven

To generate code in a given language, run `swagger-codegen generate` and pass the following
parameters:

- `--input-spec <spec file>`: this may be `https://developers.strava.com/swagger/swagger.json` or
`static/swagger/swagger.json` if you want the bleeding edge.
- `--config <configuration file>`: pass the settings or overrides you want the code generator to
honor. The overrides defined by Strava are in the `config/` directory.
- `--lang <language>`: the target programming language you seek to generate code for (running
`swagger-codegen` by itself will print a list of available languages)
- `--output <output directory>`: where to write the resulting files.

For instance:

    $ swagger-codegen generate -i developers.strava.com/swagger/swagger.json -c config/android.json -l java -o generated/java

The above will generate Java code suitable to be packaged in an Android library.

### Generating documentation

For the documentation, you need to further install `maven` – on macOS: `brew install maven`. You
then need to package the generator:

    $ mvn -f codegen/pom.xml package

After that, you need to run the `io.swagger.codegen.Codegen` class and make sure the compiled
package as well as the Swagger CLI is on the classpath — on macOS, this can be done as such:

    $ java -cp codegen/target/static-html-codegen-1.0.0.jar:$(brew --prefix swagger-codegen)/libexec/swagger-codegen-cli.jar io.swagger.codegen.Codegen -i static/swagger/swagger.json -c config/strava-html.json -l strava-html -o content/docs

This will generate the documentation in the location expected by the site.

## Content & Organization

This repository largely follows the [general layout](https://gohugo.io/overview/source-directory/)
of a Hugo site.

### Swagger definitions

The models and endpoints of the Strava API are defined as Swagger files and stored in
`static/swagger`. The individual Swagger configurations for each supported language binding are
stored in `config`.

### Documentation generator

The custom code generator for the API documentation is defined as a Maven project in the `codegen`
directory.

## Contributing