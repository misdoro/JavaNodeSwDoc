.. _XSAMSGen:

XSAMS tree generator
=========================

XSAMS is an XML schema, adopted within VAMDC for data exchange.

Java node software implementation uses JAXB library for mapping between objects and XML elements.

Contrary to the Python/Django node software, Java version doesn't provide neither it's own XML generator, nor a plain
list of mapping keywords (Returnables as they are called in Python version).
Each node plugin is responsible by itself for building object trees corresponding to the document branches and
for attaching them to the main tree.


