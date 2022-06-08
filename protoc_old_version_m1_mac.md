I was getting the following error:

Caused by: org.gradle.internal.resolve.ArtifactNotFoundException: Could not find 
  protoc-3.11.0-osx-aarch_64.exe (com.google.protobuf:protoc:3.11.0).
Searched in the following locations:
    https://repo.maven.apache.org/maven2/com/google/protobuf/protoc/3.11.0/protoc-3.11.0-osx-aarch_64.exe

Our Gradle project is in a mono-repo with getting on for a dozen sub-projects.
Several of these use protobuf code which is declared in the api module.

During compilation of these on the M1 you can see that the protobuf plugin is 
looking for a version of protoc 3.11 that is for aarch_64.  Yes there is no
version of protoc that old for aarch64 but it highlighted that the intended version
of 3.18.1 was not being picked up by all the projects.  A version does exist
for latter versions of protoc.

Solution was to ensure that the right version of protoc was set in the 
build.gradle.kts for the api for each of the sub-projects that were using protoc.

import com.google.protobuf.gradle.protobuf
import com.google.protobuf.gradle.protoc
import dependencies.*

dependencies {
    implementation(KafkaClients.dependency)
    api(project(":protobuf-mapping"))

    testImplementation(AssertJ.dependency)
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.18.1"
    }
}
