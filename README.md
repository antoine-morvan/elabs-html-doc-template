# elabs-html-doc-template
Template HTML for documentation generated from Markdown.

## Overview

This repo gives a simple HTML template for having pretty documentation pages. It consists of a ```template.html``` file where hashtags are inserted, to be replaced with the final content. Here is the list of hashtags :

 - **\# CONTENT** : the main content of the documentation page, that is under ```<body><div class="container">```;
 - **\# LANG** : the language used in the page, to appear in ```<html lang="...">```;
 - **\# TITLE** : the page title, to appear in the ```<title>``` tag, as well as in the page's header;
 - **\# AUTHOR** : the author name to appear in the ```<meta name="author" ...>``` tag;
 - **\# DESCRIPTION** : a short description of the page, to appear in the ```<meta name="description" ...>``` tag;
 - **\# COPYRIGHT** : copyright, to appear in the page footer;
 - **\# CONTACT** : contact information, to appear in the page footer;
 - **\# RELATIVE_PATH** : the relative path of the documentation root folder in the hierarchy (i.e., for a file ```/doc/basics/overview.htlm```, the relative path to its root would be ```../../```).

## Usage

Simply clone the repo somewhere in the build repository, and make your build script use the template to decorate your generated HTML pages. Here is a gradle buildscript example :

```gradle
// load git plugin
plugins {
	id 'org.ajoberstar.grgit' version '1.5.1'
}

/* 
 * note : several variables come from the gradle.properties file : 
 * version, lang, documentationTitle, author, contact
 */

apply plugin: 'org.kordamp.gradle.markdown'

// HTML output folder
def docsDir = file("$buildDir/docs")

// markdown source folder
markdownToHtml.sourceDir = file("src/main/doc")

markdownToHtml.outputDir = docsDir
markdownToHtml.configuration = [tables: true]

def description = documentationTitle
def year = new Date().format('yyyy')
def copyright = "$rootProject.name-$version &#8208; &copy; $year $author"

ext.htmlTemplateCacheDir = "$rootProject.rootDir/.htmlTemplateCache/"

/* Clone template git repo into cache */
import org.ajoberstar.grgit.*
task cloneGitRepo << {
	def destinationDir = htmlTemplateCacheDir
	def gitRepoUri = "https://github.com/echoes-tech/elabs-html-doc-template.git"
	logger.info "-- cloning html template 'gitRepoUri' into 'destinationDir'"
	if (! new File(destinationDir).exists()) {
		Grgit.clone(dir: destinationDir, uri: gitRepoUri)
	} else {
		println "-- destination folder already exists : cache is already populated."
	}
}

task buildDocumentation << {
  copy {
	from file(htmlTemplateCacheDir)
	into docsDir
  }
  def templateFile = new File(docsDir.path+"/template.html")
  if (templateFile.exists()) {
	def template = templateFile.getText("UTF-8")
	FileTree tree = fileTree(dir: "$docsDir", include: '**/*.html', exclude: ['/javadoc', '/groovydoc', '/template.html'])
	tree.each {File file ->
	  if (file.name.endsWith(".html") && !templateFile.name.equals(file.name)) {
		logger.info "-- processing markdown documentation file "+file.name+"..."
		def content = file.getText("UTF-8")
		def result = template
			.replaceAll('#CONTENT') { content }
			.replaceAll('#LANG') { lang }
			.replaceAll('#TITLE') { documentationTitle }
			.replaceAll('#AUTHOR') { author }
			.replaceAll('#DESCRIPTION') { description }
			.replaceAll('#CONTACT') { contact }
			.replaceAll('#COPYRIGHT') { copyright }
		
		// compute the relative path to the root directory
		def relativePathToParent = file.getParentFile().toPath().relativize( docsDir.toPath() ).toFile().toString()
		if (! relativePathToParent.isEmpty()) {
			relativePathToParent += "/"
		}
		result = result.replaceAll('#RELATIVE_PATH') { relativePathToParent }
		
		file.write(result, 'UTF-8')
	  }
	}
	//remove unecessary files
	templateFile.delete()
	new File(docsDir.path+"/README.md").delete()
	new File(docsDir.path+"/LICENSE").delete()
  }
}

buildDocumentation.dependsOn cloneGitRepo
buildDocumentation.dependsOn markdownToHtml

/* possible distribution extension:

apply plugin: 'distribution'

distZip.dependsOn buildDocumentation
distTar.dependsOn buildDocumentation
distributions {
  main {
	contents {
	  from docsDir
	}
  }
} */
```
