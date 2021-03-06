# Karhu node.js logging

Karhu is a simple logging library for node.js. The main feature that sets it apart is the ability
to specify different log levels for different modules, so that you don't need to see the debug
logs of the entire code base just to find the thing you are interested in.

## Installation

`npm install --save karhu`

## Usage

### Basic usage

Let's start with a simple example

    const log = require('karhu').context('app')
    // ...
    log.info('Server startup complete')
    log.error(new Error('Startup failed'))
    
### es5 compatibility

If you need es5-compatible version of the modules, you can access karhu via `require('karhu/lib/karhu-es5)`
and the default config via `require('karhu/config/default-e5')`. You might not have as much
success as you'd like though, as the `ansi-styles` dependency relies on arrow functions at the very least.

Still, if you stub that out then you can get karhu running in IE11. 

### API

For all the code examples presented here, `const karhu = require('karhu')` is expected to
be present in the file before the examples.

#### Creating a logger

By default a karhu logger is set up with the default build in configuration. Your application
can set up a different configuration to be used.

Changing the default config (which should be used by libraries using karhu) can be done by

    karhu.configure(newConfig) 

where newConfig of course is the new configuration.

You can also create karhu loggers with specific configuration objects without changing the default
by calling
   
    const configuredKarhu = karhu.usingConfig(newConfig)
    
You can then use `configuredKarhu` as if it was karhu itself, is just uses a different set of
configuration options.

Every karhu logger has context. This allows for making events from different sources distinctly
different. The context is a string; you can specify whatever you want.
 
    const log = karhu.context('mymodulename')

### Updating configuration

If you want to update configuration, you can call `reconfigure` on any karhu instance, including
the default one. All existing contexts using that instance will use the new configuration from
then on.

Log levels cannot be updated with this API at this time. 

    karhu.reconfigure({defaultLogLevel: 'DEBUG'})

### Wrapping output outside of explicit logging

You may want to enforce a consistent format to all output, even if it is done by, let's say
a library that uses `console.log`.

This is something karhu can help you with. By calling `captureStandardOutput` you set up
capturing of standard output and turning that into log events.

By default everything to stdout becomes `INFO` events and everything to stderr becomes `ERROR`
events.

The function must be provided with the context it will use.

    karhu.captureStandardOutput(karhu.context('captured-output'), 'INFO', 'ERROR')
    
If you for whatever reason want to stop capturing the output, you can call `abort()` on the
value returned by the call

    const abortable = karhu.captureStandardOutput(karhu.context('captured-output'))
    abortable.abort()

### Logging

A logger has methods for all of the specified logging levels. The default levels are

- none; never to be printed; not useful for logging, but can be used to disable logging
- debug
- info
- warn
- error

Each method can take any number of all kinds of arguments and they will (by default) be printed
just as `console.log` would.        

### Configuration

Karhu is configured using a javascript object, that has properties presented below. You probably want
to base your config based on the default config, which can be obtained using `require('karhu/config/default')`

#### logLevels

`logLevels` is an array of strings, which are the log levels available for logging. The log levels in this array
should generally be in ALL CAPS, they will be available in the logger both using the presented form as well
as in all lowercase. In other words, `ERROR` here allows both `log.error` and `log.ERROR` to be used. 

If multiple log levels map to the same lowercase form (and neither is all lowercase), then the first one encountered
will be made available.

The log levels must be presented in ascending order of importance.

    const logLevels = ['INF', 'WARN', 'ERR']

#### colors

Karhu supports colors to emphasize more important information. `colors` is an object that maps log levels to their
colors. Colors are expected to be colors from the `ansi-styles` node.js library, or compatible.

A single color can be defined, or an array of colors if you want to change both the foreground and background colors, 
for example.

    const colors = {
        ERROR: [styles.color.whiteBright, styles.bgColor.redBright],
        NOTICE: styles.color.blueBright
    }

You can also specify color for `default`, which is used if no other color matches.

#### outputFormat

The format that is to be used for output. The default config includes support for `text` and `json`,
but you can specify additional formats in `formatters`. The default config defaults to `text`, unless
the environment variable `KARHU_JSON` is set to a truthy value.

#### formatters

Formatters transform the logged data into something that can be output. Typically that means
strings, but anything accepted by the `transports` of the config can be used.

The default config includes formatters for `text` and `json`.

#### formatNow

This option should contain a function that returns the current moment timestamp in whatever
format is desired for the log.

The default config uses `new Date().toISOString()`, which produces a timestamp that looks
pretty much like this: `2018-02-08T11:20:25.690Z`, always in UTC.

The default formatters utilize this function, but custom formatters might not.

    const formatNow = config => new Date().toISOString()
    
The function gets the active configuration as a parameter, in case you want to make the decision
based on output format, for example.

#### contextSpecificLogLevels

This map or object allows you to specify log levels for individual contexts to be different from the
default.

The context can be either the specific context as a string or a regular expression that will
apply the log level to any matching context. Specific log level on a context always
overrides regular expressions.

    const contextSpecificLogLevels = new Map([
        ["src/app", "ERROR"], // ignore everything less than an error
        [/special/, "DEBUG"]
    ])
    
    const contextSpecificLogLevels = {
        "src/app": "ERROR"
    }
    
Regular expressions are not supported when the object form is used.
    
#### defaultLogLevel

This string specifies the default log level -- anything lower in importance will not be logged.

#### envVariablePrefix

This variable can be used to specify a prefix for environment variables used for runtime configuration.
By default the prefix is just `KARHU`.

#### outputMapper

This is a function that maps data being output to another form. It is given a value being logged
along with other relevant information and it is expected to return whatever is to be passed to
`console.log` (or one of its siblings).

    const outputMapper = (value, logLevel, context, toLog) => {
        if (value === undefined) return '(undefined)'
        return value    
    } 

#### transports

Transports is an object hash from transport name to means of generating the output. A transport can override most
of the settings from the top level config, inheriting everything not overridden.

    const transports = {
        console: consoleTransport     
    }
    
The default implementation includes `console` transport, which calls `console.error` for 
errors, `console.warn` for warning and `console.log` for everything else.

The sections below describe the transport options not available on top level config

##### supportsColor

A boolean or function that is to return a boolean set to true if (ANSI) colors are supported by the transport.

    const supportsColor = () => false

##### outputImpl

    const consoleTransport = {
         ERROR: (toLog: any[]) => console.error(toLog),
         WARN: (toLog: any[]) => console.warn(toLog), 
         default: (toLog: any[]) => console.log(toLog)
    ])

    const outputImpl = {console: consoleTransport}
    
The functions receive the things being logged as an array (after applying mapping of `outputMapper`),
the log level and context for the event and the active configuration.

You can specify a different methdod for each log level, and optionally a `default` which will be used
whenever there is no function specified for the one being used. 

### Run-time configuration

You can override some configuration options using environment variables.

Note that the configuration can change the KARHU prefix of the environment variables to become
something else.

#### Overriding default log level

You can set a different default log level to be used using the `KARHU_LOG_LEVEL` environment
variable.

    KARHU_LOG_LEVEL=debug node src/app

#### Overriding context-specific log level

You can set a different log level for an individual module using environment variables that
include the context.

    KARHU_LOG_LEVEL_src_app=DEBUG node src/app # set DEBUG as the log level for src/app
    
Any sequences for non-English-alphanumeric characters in the module name are replaced by
underscores in the environment variable name. 

## Development

### Running tests

    npm test
    
### Releasing

    npm release patch # +0.+0.+1
    npm release minor # +0.+1.0
    npm release major # +1.0.0
