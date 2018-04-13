# Safe pipe fork of Apache HTTP Server

If for some reason you don't know what the Apache HTTP Server is, see below.
The HTTP server is what "started it all".

This fork addresses a long-standing limitation of log record sizes when using
piped logs ([reported here](https://bz.apache.org/bugzilla/show_bug.cgi?id=54339)).
With piped logs (`|` option), log records are piped to a separate program that
can batch/rotate log records or do whatever your heart desires with them.
A popular use-case is having `rotatelogs` receive the log records for flexible
log rotation without having to HUP the server, etc.

Depending on your OS, there may be an atomic buffer limit for data sent over a pipe,
beyond which, data may be interleaved (e.g. 4KB for Linux). What this means is
that if you have multiple writers to the pipe, the output may be corrupted.
With apache, you most certainly will have concurrent writers to the pipe, so
if your log records exceed this limit, your log files will be sawdust. To be
fair, most sites will not be affected as typical log records
are well under the pipe buffer limit (PIPE_BUF on Linux). If you find yourself
approaching an edge case, this fix is for you.

### Usage

#### Patch Apache (merge the fix)

* Use the `ASF-54339-interleaved-logs-2.4.33` branch to patch v2.4.
* Or if you're on the bleeding edge, check out the `ASF-54339-interleaved-logs-2.5-HEAD` branch.

#### Log configuration

For piped log streams in your configuration that use `rotatelogs`,
where log records are extra wide, replace `|` with `|<` (mnemonic, split pipe).
This will split log records across the pipe into smaller chunks (within the
atomic size guarantee) so they may be reassembled by the piped
program to preserve their integrity. The program receiving the
records must know how to reassemble the records, and currently,
rotatelogs supports this. When used, a variable
(AP_MOD_LOG_CONFIG_CHUNKED_MSG) is exported to the environment
so the receiving program can take appropriate action. This option
may be used in conjunction with the shell option `|<$` (or for
compatibility `||<`).
This option is not available for ErrorLog directives and if
present, a warning will be emitted.

Example:
```
CustomLog "|</usr/local/apache/bin/rotatelogs   /var/log/access_log 86400" common
```


```
                          Apache HTTP Server

  What is it?
  -----------

  The Apache HTTP Server is a powerful and flexible HTTP/1.1 compliant
  web server.  Originally designed as a replacement for the NCSA HTTP
  Server, it has grown to be the most popular web server on the
  Internet.  As a project of the Apache Software Foundation, the
  developers aim to collaboratively develop and maintain a robust,
  commercial-grade, standards-based server with freely available
  source code.

  The Latest Version
  ------------------

  Details of the latest version can be found on the Apache HTTP
  server project page under http://httpd.apache.org/.

  Documentation
  -------------

  The documentation available as of the date of this release is
  included in HTML format in the docs/manual/ directory.  The most
  up-to-date documentation can be found at
  http://httpd.apache.org/docs/trunk/.

  Installation
  ------------

  Please see the file called INSTALL.  Platform specific notes can be
  found in README.platforms.

  Licensing
  ---------

  Please see the file called LICENSE.

  Cryptographic Software Notice
  -----------------------------

  This distribution may include software that has been designed for use
  with cryptographic software.  The country in which you currently reside
  may have restrictions on the import, possession, use, and/or re-export
  to another country, of encryption software.  BEFORE using any encryption
  software, please check your country's laws, regulations and policies
  concerning the import, possession, or use, and re-export of encryption
  software, to see if this is permitted.  See <http://www.wassenaar.org/>
  for more information.

  The U.S. Government Department of Commerce, Bureau of Industry and
  Security (BIS), has classified this software as Export Commodity 
  Control Number (ECCN) 5D002.C.1, which includes information security
  software using or performing cryptographic functions with asymmetric
  algorithms.  The form and manner of this Apache Software Foundation
  distribution makes it eligible for export under the License Exception
  ENC Technology Software Unrestricted (TSU) exception (see the BIS 
  Export Administration Regulations, Section 740.13) for both object 
  code and source code.

  The following provides more details on the included files that
  may be subject to export controls on cryptographic software:

    Apache httpd 2.0 and later versions include the mod_ssl module under
       modules/ssl/
    for configuring and listening to connections over SSL encrypted
    network sockets by performing calls to a general-purpose encryption
    library, such as OpenSSL or the operating system's platform-specific
    SSL facilities.

    In addition, some versions of apr-util provide an abstract interface
    for symmetrical cryptographic functions that make use of a
    general-purpose encryption library, such as OpenSSL, NSS, or the
    operating system's platform-specific facilities. This interface is
    known as the apr_crypto interface, with implementation beneath the
    /crypto directory. The apr_crypto interface is used by the
    mod_session_crypto module available under
      modules/session
    for optional encryption of session information.

    Some object code distributions of Apache httpd, indicated with the
    word "crypto" in the package name, may include object code for the
    OpenSSL encryption library as distributed in open source form from
    <http://www.openssl.org/source/>.

  The above files are optional and may be removed if the cryptographic
  functionality is not desired or needs to be excluded from redistribution.
  Distribution packages of Apache httpd that include the word "nossl"
  in the package name have been created without the above files and are
  therefore not subject to this notice.

  Contacts
  --------

     o If you want to be informed about new code releases, bug fixes,
       security fixes, general news and information about the Apache server
       subscribe to the apache-announce mailing list as described under
       <http://httpd.apache.org/lists.html#http-announce>

     o If you want freely available support for running Apache please see the
       resources at <http://httpd.apache.org/support.html>

     o If you have a concrete bug report for Apache please see the instructions
       for bug reporting at <http://httpd.apache.org/bug_report.html>

     o If you want to participate in actively developing Apache please
       subscribe to the `dev@httpd.apache.org' mailing list as described at
       <http://httpd.apache.org/lists.html#http-dev>

```
