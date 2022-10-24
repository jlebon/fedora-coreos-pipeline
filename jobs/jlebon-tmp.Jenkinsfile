import org.yaml.snakeyaml.Yaml;

def pipeutils, pipecfg, official, uploading, libupload
node {
    checkout scm
    pipeutils = load("utils.groovy")
    pipecfg = pipeutils.load_pipecfg()
    libupload = load("libupload.groovy")

    def jenkinscfg = pipeutils.load_jenkins_config()

    official = pipeutils.isOfficial()
    if (official) {
        echo "Running in official (prod) mode."
    } else {
        echo "Running in unofficial pipeline on ${env.JENKINS_URL}."
    }
}

println(pipeutils.get_artifacts_to_build(pipecfg, "testing-devel", "x86_64"))
println(pipeutils.get_artifacts_to_build(pipecfg, "testing-devel", "aarch64"))
println(pipeutils.get_artifacts_to_build(pipecfg, "testing-devel", "ppc64le"))
println(pipeutils.get_artifacts_to_build(pipecfg, "testing-devel", "s390x"))
