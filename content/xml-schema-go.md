+++
date = "2015-02-22T21:31:15-05:00"
title = "XML Schema and Go"
description = "Generating Go source from XML schema"
tags = ["xml", "go", "soap"]
+++

Like it or hate it, XML is a reality that many of us have to deal
with on a daily basis at the workplace. The `encoding/xml` package
in Go's standard library provides a convenient, data-driven approach
to parsing XML documents that is usually sufficient for most use
cases. When you are dealing with a massive API, however, it quickly
grows tedious translating XML structures to Go types.

Most large SOAP-based web services provide a formal description of
the XML structures they use in the form of [XML Schema][0]. The XML
Schema standard is very large, and its 2-part [spec][1] is written
in highly abstract, difficult language. I find it amusing that both
the specification and XML schema documents themselves are full of
boilerplate.

Languages with strong support for XML-based services, such as #C
and Java, have very rich code-generation tools, that let you generate
source code for working with the XML elements described in an XML
Schema. The [xsdgen][2] package is my attempt to add Go to that
list. With the [go generate][3] feature, added in Go 1.4, code
generation is easier than ever.

## Generating Go types from XML Schema

We have IPAM software at my workplace that provides a SOAP API. It has the following schema (anonymized) in its wsdl file:

	<schema targetNamespace="http://example.com/"
		xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/"
		xmlns:xsd="http://www.w3.org/2001/XMLSchema"
		xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/"
		xmlns:tns="http://example.com/"
		xmlns="http://www.w3.org/2001/XMLSchema">
	 <complexType name="WSDevice">
	  <sequence>
	   <element name="addressType" nillable="true" type="soapenc:string"/>
	   <element name="description" nillable="true" type="soapenc:string"/>
	   <element name="deviceType" nillable="true" type="soapenc:string"/>
	   <element name="domainName" nillable="true" type="soapenc:string"/>
	   <element name="hostname" nillable="true" type="soapenc:string"/>
	   <element name="id" nillable="true" type="soapenc:int"/>
	   <element maxOccurs="unbounded" name="interfaces" nillable="true" type="tns1:WSInterface"/>
	   <element name="ipAddress" nillable="true" type="soapenc:string"/>
	  </sequence>
	 </complexType>
	 <complexType name="WSInterface">
	  <sequence>
	   <element name="id" nillable="true" type="soapenc:int"/>
	   <element name="ipAddress" nillable="true" type="tns:ArrayOf_soapenc_string"/>
	   <element name="macAddress" nillable="true" type="soapenc:string"/>
	   <element name="name" nillable="true" type="soapenc:string"/>
	   <element name="sequence" nillable="true" type="soapenc:int"/>
	   <element name="virtual" nillable="true" type="soapenc:boolean"/>
	  </sequence>
	 </complexType>
	 <complexType name="ArrayOf_soapenc_string">
	  <complexContent>
	   <restriction base="soapenc:Array">
	    <attribute ref="soapenc:arrayType" wsdl:arrayType="soapenc:string[]"/>
	   </restriction>
	  </complexContent>
	 </complexType>
	</schema>

The [xsdgen][4] command is suitable for use with `go generate`. In
my workspace, I save the wsdl file as "schema.xml" and created the
file `gen.go` with the following lines:

	package ipam
	
	//go:generate xsdgen -ns http://example.com/ -pkg ipam schema.xml

Running "go generate" produces the file "xsdgen_output.go":

	package ipam
	
	import "encoding/xml"
	
	type ArrayOfsoapencstring []string
	
	func (a *ArrayOfsoapencstring) MarshalXML(e *xml.Encoder, start xml.StartElement) error {
		tag := xml.StartElement{Name: xml.Name{"", "item"}}
		for _, elt := range *a {
			if err := e.EncodeElement(elt, tag); err != nil {
				return err
			}
		}
		return nil
	}
	func (a *ArrayOfsoapencstring) UnmarshalXML(d *xml.Decoder, start xml.StartElement) (err error) {
		var tok xml.Token
		var itemTag = xml.Name{"", ",any"}
		for tok, err = d.Token(); err == nil; tok, err = d.Token() {
			if tok, ok := tok.(xml.StartElement); ok {
				var item string
				if itemTag.Local != ",any" && itemTag != tok.Name {
					err = d.Skip()
					continue
				}
				if err = d.DecodeElement(&item, &tok); err == nil {
					*a = append(*a, item)
				}
			}
			if _, ok := tok.(xml.EndElement); ok {
				break
			}
		}
		return err
	}
	
	type WSDevice struct {
		AddressType string        `xml:"http://example.com/ addressType"`
		Description string        `xml:"http://example.com/ description"`
		DeviceType  string        `xml:"http://example.com/ deviceType"`
		DomainName  string        `xml:"http://example.com/ domainName"`
		Hostname    string        `xml:"http://example.com/ hostname"`
		Id          int           `xml:"http://example.com/ id"`
		Interfaces  []WSInterface `xml:"http://example.com/ interfaces"`
		IpAddress   string        `xml:"http://example.com/ ipAddress"`
	}
	type WSInterface struct {
		Id         int                  `xml:"http://example.com/ id"`
		IpAddress  ArrayOfsoapencstring `xml:"http://example.com/ ipAddress"`
		MacAddress string               `xml:"http://example.com/ macAddress"`
		Name       string               `xml:"http://example.com/ name"`
		Sequence   int                  `xml:"http://example.com/ sequence"`
		Virtual    bool                 `xml:"http://example.com/ virtual"`
	}

I can replace ugly names by modifying my xsdgen command:

	//go:generate xsdgen -ns http://example.com/ -r "WS(.*) -> $1" -r "ArrayOf_soapenc_string -> Strings" -pkg ipam schema.xml

Will produce types named `Device`, `Strings`, `Interface`. The
xsdgen package respects xml namespaces and inheritance; it knows
that a `soapenc:string` is derived from an `xsd:string`, for instance.
Rather than preserving this hierarchy in the generated Go source,
the xsdgen package "squashes" all inheritence, and tries to minimize
the levels of indirection between any given type and the builtin
types defined in the XML schema specification. This is done to
reduce the amount of code generated and provide a more pleasant
experience for the user of the generated library. While writing
these packages I became acutely aware of how heavily XML Schema was
influenced by inheritence-ridden OOP languages such as Java.

## Customizing the behavior of xsdgen

You may need to customize the code generation process more than
what the command-line flags to xsdgen allow. For instances, say that
you do not care about the "sequence" or "virtual" elements defined in
the schema above.

Create the file `_gencfg/cfg.go`. The name is not important. I prefix
the directory with an underscore so that commands such as `go build ./...`
ignore it. The file contains something like this:

	package main
	
	import (
		"log"
		"os"
	
		"aqwari.net/xml/xsdgen"
	)
	
	func main() {
		var cfg xsdgen.Config
		cfg.Option(xsdgen.DefaultOptions...)
		cfg.Option(
			xsdgen.LogOutput(log.New(os.Stderr, "", 0)),
			xsdgen.IgnoreElements("virtual", "sequence"))
		
		if err := cfg.GenCLI(os.Args[1:]...); err != nil {
			log.Fatal(err)
		}
	}

The full set of Options available can be found in the documentation for
the [xsdgen package][2]. Some Options are pretty advanced, and give
provide a shim for manipulating types and Go syntax trees with arbitrary
code. Once this file is created, update the `gen.go` file:

	//go:generate go run _gencfg/cfg.go -ns http://example.com/ -r "^WS ->" -r "ArrayOfsoapencstring -> Strings" -pkg ipam schema.xml

The declaration of `Interface` then becomes

	type Interface struct {
		Id         int     `xml:"http://example.com/ id"`
		IpAddress  Strings `xml:"http://example.com/ ipAddress"`
		MacAddress string  `xml:"http://example.com/ macAddress"`
		Name       string  `xml:"http://example.com/ name"`
	}

This was my first time using the `go/ast` package in the Go standard
library. I recommend that anyone doing non-trivial code generation
look at using `go/ast` instead of `text/template`; being able to 
manipulate expressions as data structures is very powerful. For instance,
a SOAP array is naiively mapped to the structure

	type Array struct {
		Items []T `xml:",any"`
	}

As a post-processing step, the xsdgen package looks for any structures
that contain a single slice element, and changes the type expression to

	type Array []T

Because it uses an `*ast.StructType` instead of opaque text, the code
can reach in and access information such as struct tags for use in
marshal/unmarshal methods.

## Making it better

The code for the `xsdgen` and related packages is on [github][5].

[0]: http://www.w3.org/TR/xmlschema-0/
[1]: http://www.w3.org/TR/xmlschema-1/
[2]: http://aqwari.net/xml/xsdgen/
[3]: http://blog.golang.org/generate
[4]: http://aqwari.net/xml/cmd/xsdgen/
