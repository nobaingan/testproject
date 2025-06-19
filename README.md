configurations {
    jaxb
    xjc
}

dependencies {
    // JAXB Jakarta dependencies
    jaxb 'org.glassfish.jaxb:jaxb-xjc:3.0.2'
    jaxb 'org.glassfish.jaxb:jaxb-runtime:3.0.2'
    jaxb 'org.glassfish.jaxb:jaxb-core:3.0.2'
    jaxb 'jakarta.xml.bind:jakarta.xml.bind-api:3.0.1'
    jaxb 'com.sun.activation:jakarta.activation:2.0.1'

    // JAXB Plugins (ensure they're JAXB 3 compatible)
    xjc 'org.jvnet.jaxb2_commons:jaxb2-basics:0.12.0'
    xjc 'org.jvnet.jaxb2_commons:jaxb2-basics-annotate:1.1.1' // Updated for JAXB 3
    // Optional: Remove this if it causes JAXB 3 incompatibility
    // xjc 'com.github.jaxb-xew-plugin:jaxb-xew-plugin:1.11'

    xjc 'commons-beanutils:commons-beanutils:1.9.4'
    xjc 'commons-io:commons-io:2.17.0'
    xjc 'org.projectlombok:lombok'
}

task generateJaxb(type: JavaExec) {
    group = 'build'
    description = 'Generates Java sources from XSD using JAXB 3.x'
    mainClass.set('com.sun.tools.xjc.XJCFacade')
    classpath = configurations.jaxb + configurations.xjc

    def schemaDir = file('src/main/resources/schemas/services/Authentic')
    def outputDir = "$buildDir/generated-sources/jaxb"

    inputs.files fileTree(dir: schemaDir, include: '*.xsd')
    outputs.dir outputDir

    args = [
        '-d', outputDir,
        '-p', 'com.authentic',
        '-extension',
        '-b', "$schemaDir/bindings.xjb",
        '-Xannotate',
        '-Xinheritance'
        // If xew plugin is compatible, uncomment the line below
        // '-Xxew'
    ] + fileTree(schemaDir).matching { include '*.xsd' }.files*.absolutePath
}

sourceSets.main.java.srcDir "$buildDir/generated-sources/jaxb"
