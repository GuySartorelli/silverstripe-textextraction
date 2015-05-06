# Text Extraction Module

[![Build Status](https://secure.travis-ci.org/silverstripe-labs/silverstripe-textextraction.png)](http://travis-ci.org/silverstripe-labs/silverstripe-textextraction)

## Overview

Provides an extraction API for file content, which can hook into different extractor
engines based on availability and the parsed file format.
The output is always a string: the file content.

Via the `FileTextExtractable` extension, this logic can be used to 
cache the extracted content on a `DataObject` subclass (usually `File`).

Note: Previously part of the [sphinx module](https://github.com/silverstripe/silverstripe-sphinx).

## Requirements

 * SilverStripe 3.1
 * (optional) [XPDF](http://www.foolabs.com/xpdf/) (`pdftotext` utility)
 * (optional) [Apache Solr with ExtracingRequestHandler](http://wiki.apache.org/solr/ExtractingRequestHandler)
 * (optional) [Apache Tika](http://tika.apache.org/)

### Supported Formats

 * HTML (built-in)
 * PDF (with XPDF or Solr)
 * Microsoft Word, Excel, Powerpoint (Solr)
 * OpenOffice (Solr)
 * CSV (Solr)
 * RTF (Solr)
 * EPub (Solr)
 * Many others (Tika)

## Installation

The recommended installation is through [composer](http://getcomposer.org).
Add the following to your `composer.json`:

```js
{
	"require": {
		"silverstripe/textextraction": "2.0.x-dev"
	}
}
```

The module depends on the [Guzzle HTTP Library](http://guzzlephp.org),
which is automatically checked out by composer. Alternatively, install Guzzle
through PEAR and ensure its in your `include_path`.

## Configuration

### Basic

By default, only extraction from HTML documents is supported.
No configuration is required for that, unless you want to make
the content available through your `DataObject` subclass.
In this case, add the following to `mysite/_config/config.yml`:

```yaml
File:
  extensions:
	- FileTextExtractable
```

By default any extracted content will be cached against the database row.

Alternatively, extracted content can be cached using SS_Cache to prevent excessive database growth.
In order to swap out the cache backend you can use the following yaml configuration.


```yaml
---
Name: mytextextraction
After: '#textextraction'
---
Injector:
  FileTextCache: FileTextCache_SSCache
FileTextCache_SSCache:
  lifetime: 3600 # Number of seconds to cache content for

```

### XPDF

PDFs require special handling, for example through the [XPDF](http://www.foolabs.com/xpdf/)
commandline utility. Follow their installation instructions, its presence will be automatically
detected. You can optionally set the binary path in `mysite/_config/config.yml`:

```yml
PDFTextExtractor:
	binary_location: /my/path/pdftotext
```

### Apache Solr

Apache Solr is a fulltext search engine, an aspect which is often used
alongside this module. But more importantly for us, it has bindings to [Apache Tika](http://tika.apache.org/)
through the [ExtractingRequestHandler](http://wiki.apache.org/solr/ExtractingRequestHandler) interface.
This allows Solr to inspect the contents of various file formats, such as Office documents and PDF files.
The textextraction module retrieves the output of this service, rather than altering the index.
With the raw text output, you can decide to store it in a database column for fulltext search
in your database driver, or even pass it back to Solr as part of a full index update.

In order to use Solr, you need to configure a URL for it (in `mysite/_config/config.yml`):

```yml
SolrCellTextExtractor:
	base_url: 'http://localhost:8983/solr/update/extract'
```

Note that in case you're using multiple cores, you'll need to add the core name to the URL 
(e.g. 'http://localhost:8983/solr/PageSolrIndex/update/extract').
The ["fulltext" module](https://github.com/silverstripe-labs/silverstripe-fulltextsearch)
uses multiple cores by default, and comes prepackaged with a Solr server.
Its a stripped-down version of Solr, follow the module README on how to add
Apache Tika text extraction capabilities.

You need to ensure that some indexable property on your object
returns the contents, either by directly accessing `FileTextExtractable->extractFileAsText()`,
or by writing your own method around `FileTextExtractor->getContent()` (see "Usage" below).
The property should be listed in your `SolrIndex` subclass, e.g. as follows:

```php
class MyDocument extends DataObject {
	static $db = array('Path' => 'Text');
	function getContent() {
		$extractor = FileTextExtractor::for_file($this->Path);
		return $extractor ? $extractor->getContent($this->Path) : null;		
	}
}
class MySolrIndex extends SolrIndex {
	function init() {
		$this->addClass('MyDocument');
		$this->addStoredField('Content', 'HTMLText');
	}
}
```

Note: This isn't a terribly efficient way to process large amounts of files, since 
each HTTP request is run synchronously.

### Tika

Support for Apache Tika (1.8 and above) is included. This can be run in one of two ways: Server or CLI.

See [the Apache Tika home page](http://tika.apache.org/1.8/index.html) for instructions on installing and
configuring this. Download the latest `tika-app` for running as a CLI script, or `tika-server` if you're planning
to have it running constantly in the background. Starting tika as a CLI script for every extraction request
is fairly slow, so we recommend running it as a server.

This extension will best work with the [fileinfo PHP extension](http://php.net/manual/en/book.fileinfo.php)
installed to perform mime detection. Tika validates support via mime type rather than file extensions.

### Tika - CLI

Ensure that your machine has a 'tika' command available which will run the CLI script.

```bash
#!/bin/bash
exec java -jar tika-app-1.8.jar "$@"
```

### Tika Rest Server

Tika can also be run as a server. You can configure your server endpoint by setting the url via config.

```yaml
TikaServerTextExtractor:
  server_endpoint: 'http://localhost:9998'
```

Alternatively this may be specified via the `SS_TIKA_ENDPOINT` directive in your `_ss_environment.php` file, or an environment variable of the same name.


Then startup your server as below

```bash
java -jar tika-server-1.8.jar --host=localhost --port=9998
```

While you can run `tika-app-1.8.jar` in server mode as well (with the `--server` flag),
it behaves differently and is not recommended.

The module will log extraction errors with `SS_Log::NOTICE` priority by default,
for example a "422 Unprocessable Entity" HTTP response for an encrypted PDF.
In case you want more information on why processing failed, you can increase
the logging verbosity in the tika server instance by passing through
a `--includeStack` flag. Logs can passed on to files or external logging services,
see [error handling](http://doc.silverstripe.org/en/developer_guides/debugging/error_handling)
documentation for SilverStripe core.

## Usage

Manual extraction:

```php
$myFile = '/my/path/myfile.pdf';
$extractor = FileTextExtractor::for_file($myFile);
$content = $extractor->getContent($myFile);
```

Extraction with `FileTextExtractable` extension applied:

```php
$myFileObj = File::get()->First();
$content = $myFileObj->getFileContent();
```

This content can also be embedded directly within a template.

```
$MyFile.FileContent
```
