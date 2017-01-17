---
layout: post
title: "Map to String and Back"
date: 2015-12-18 15:27:52 +1100
comments: true
categories: 
---




`ramsRex = """\(([^)]+)\)""".r
    val params = Map("a" -> 1, "b" -> 2).toString

    // Turn a string Map(a -> 1, b -> 2) into a Map object
    val paramMap = paramsRex.findFirstMatchIn(params).map { mtch =>
      mtch.group(1).split(", ").map { entry =>
        val kv = entry.split(" -> ")
        (kv(0), kv(1))
      }
    } getOrElse(Array()) toMap


    val OLDOONE = paramsRex.findFirstMatchIn(params).map { mtch =>
        val entries = mtch.group(1).split(", ")
        entries.map { e =>
          val kv = e.split(" -> ")
          (kv(0), kv(1))
        }
      } getOrElse(Array()) toMap

    //
    //    //flatmap - map - map
    val z = paramsRex.findFirstMatchIn(params).flatMap { mtch =>
      Option(mtch.group(1).split(", "))
    } map{ entryArr =>
      entryArr.map { e =>
        val kv = e.split(" -> ")

        (kv(0), kv(1))
      }
    }


    //For comprehension 1
    val x = {
      for {
        mtch <- paramsRex.findFirstMatchIn(params)
        entryArr <- Option(mtch.group(1).split(", "))
      } yield {
        entryArr.map { entry =>
          val kv = entry.split(" -> ")

          (kv(0), kv(1))
        }
      }
    }

    //For comprehension 2
    val zx = {
      paramsRex.findFirstMatchIn(params).map { mtch =>
        for {
          entry <- mtch.group(1).split(", ")
          kv = entry.split(" -> ")
        } yield {
          (kv(0), kv(1))
        }
      }
    }

    //Interative - More efficient??
    val res = scala.collection.mutable.HashMap[String, String]()
    paramsRex.findFirstMatchIn(params).map { mtch =>
      mtch.group(1).split(", ").foreach { entry =>
          val kv = entry.split(" -> ")
          res.put(kv(0), kv(1))
        }
      }
    val xx = res.toMaprequire': cannot load such file -- mkmf (LoadError)dd

