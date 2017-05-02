# DataDog OpenTracing PHP

[![Build Status](https://travis-ci.org/jcchavezs/dd-opentracing-php.svg?branch=master)](https://travis-ci.org/jcchavezs/dd-opentracing-php)
[![Join the chat at https://gitter.im/dd-opentracing-php/Lobby](https://badges.gitter.im/dd-opentracing-php/Lobby.svg)](https://gitter.im/dd-opentracing-php/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)


DataDog implementation for OpenTracing in PHP

## Installation

Execute:

```sh
composer require jcchavezs/dd-opentracing
```

## Examples

```php
use DdOpenTracing\Tracer;
use OpenTracing\Carriers\HttpHeaders;
use OpenTracing\GlobalTracer;
use OpenTracing\Propagator;
use OpenTracing\SpanContext;
use OpenTracing\SpanReference\ChildOf;
use OpenTracing\Tag;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

$request = Request::createFromGlobals();

$client = new \GuzzleHttp\Client();
$logger = new \Monolog\Logger("log");
$encoderFactory = new \DdTrace\Encoders\JsonFactory;
$transport = new \DdTrace\Transports\Http($client, $logger, $encoderFactory);
$buffer = new \DdTrace\Buffer();
$ddTracer = new \DdTrace\Tracer($buffer, $logger, $transport);

$tracer = new Tracer($ddTracer, $logger);
$tracer->enableDebugLogging();

GlobalTracer::setGlobalTracer($tracer);

$spanContext = GlobalTracer::globalTracer()->extract(
    Propagator::HTTP_HEADERS,
    HttpHeaders::fromRequest($request)
);

usleep(200);

readFromDB($spanContext);

$response = Response::create("", 200);

$response->send();

// This flushes the traces, if the buffer could be persisted, a worker could flush the traces from time to time.
$tracer->tracer()->flushTraces();

function readFromDB(SpanContext $spanContext)
{
    $tracer = GlobalTracer::globalTracer();

    $component = Tag::create("component", "SELECT * FROM test_table");
    $peerService = Tag::create("peer.service", "test_service");

    $span = $tracer->startSpan("Db read", ChildOf::withContext($spanContext), null, $component, $peerService);

    usleep(3000);

    $span->finish();
}
```
