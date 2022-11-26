def pipeutils
node {
    checkout scm
    pipeutils = load("utils.groovy")
}

properties([
  parameters([
    booleanParam(name: 'REAL_PARAM', defaultValue: false)
  ])
])

if (pipeutils.triggered_by_seed()) {
    println("Triggered by seed job.")
    currentBuild.description = "[triggered by seed job] 🔄"
    return
}

echo "real job"
