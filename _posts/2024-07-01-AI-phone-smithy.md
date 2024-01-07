---
title: AI Phone Smithy
---

# Aim
Turn every smithy4s SimpleRestJson endpoint, into a tool which can be called inside openAIs function calling mechanism.

# Tools of interest
BSP
AWS API
Corporate API

# Hardness
Very

# Status
Initial implementation done, but needs a complete rewrite

I'd model the JsonSchema construct in terms of simple monomorphic data structure (case classes / ADTs). No layer of this structure would reference Hints directly. Right now you have something that's a bit hard to follow, because it's not simple data, and uses a lot of object-oriented techniques that are counter-intuitive from a reader point of view. I'd accompany each case class of that data structure with Document.Encoder (or even a manually constructed smithy4s.schema.Schema).

JsonSchemaVisitor would be something like :

```scala
type SeenShapes = Set[ShapeId]
type Defs = Map[ShapeId, JsonSchema]

// This is an intermediate data structure that helps the implementation of the SchemaVisitor
case class JsonSchemaLayer(shapeId: ShapeId, current: JsonSchema, transitiveDefs: Defs) {
   // makes it easy to turn a layer into a map after recursing
   def allDefs: Defs = transitiveDefs + (shapeId -> current)

   // when dealing with structures, you may want to use a `fold`
   def allSeen: SeenShapes = transitiveDefs.keySet + shapeId
}

// Every time you recurse, you then need to invoke the function of the current shapeId, so that
// the `lazily` method get a chance to return `None` if its ShapeId has already been encountered
type JsonSchemaBuilder[A] : SeenShapes => Option[JsonSchemaLayer]
object JsonSchemaBuilder extends SchemaVisitor[JsonSchemaBuilder] {
   // implement everything ...


   def lazily[A](schema: Schema[A]) = new (SeenShapes => Option[JsonSchemaLayer]){
       lazy val underlying = schema.compile(this)

       def apply(seenShapes: SheenShapes) : Option[JsonSchemaLayer]{
          if(seenShapes(schema.shapeId)) None
          else underlying(seenShapes)
       }
   }
}
```
