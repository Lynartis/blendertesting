fn bakeUVGradientToBlue obj uvChannel:1 =
(
    if classOf obj.baseObject != Editable_Poly do
        convertToPoly obj

    local vcChannel = 0
    local faceCount = polyop.getNumFaces obj

    -- Initialize vertex color channel
    polyop.defaultMapFaces obj vcChannel

    local processedFaces = #{}

    for f = 1 to faceCount do
    (
        if not processedFaces[f] do
        (
            -- Get complete geometry element
            local elementFaces = polyop.getElementsUsingFace obj #{f}

            processedFaces += elementFaces

            --------------------------------------------------
            -- Find UV V min/max for this element
            --------------------------------------------------

            local minV = 1e9
            local maxV = -1e9

            for faceIndex in elementFaces do
            (
                local uvFace = polyop.getMapFace obj uvChannel faceIndex

                for uvIndex in uvFace do
                (
                    local uv = polyop.getMapVert obj uvChannel uvIndex

                    if uv.y < minV do minV = uv.y
                    if uv.y > maxV do maxV = uv.y
                )
            )

            local uvRange = maxV - minV

            --------------------------------------------------
            -- Paint VC blue from bottom -> top
            --------------------------------------------------

            for faceIndex in elementFaces do
            (
                local uvFace = polyop.getMapFace obj uvChannel faceIndex
                local vcFace = polyop.getMapFace obj vcChannel faceIndex

                for corner = 1 to uvFace.count do
                (
                    local uv = polyop.getMapVert obj uvChannel uvFace[corner]

                    local gradient = 0.0

                    if uvRange > 0.000001 do
                        gradient = (uv.y - minV) / uvRange

                    gradient = amax 0.0 (amin 1.0 gradient)

                    local vcIndex = vcFace[corner]

                    polyop.setMapVert obj vcChannel vcIndex [0, 0, gradient]
                )
            )
        )
    )

    update obj
)

# blendertesting
messing around with blender

try(destroyDialog VertexBPaint)catch()

rollout VertexBPaint "Vertex B Paint"
(
    button btnEdit "Edit Vertex Color B" width:180 height:30
    button btnSave "Save Vertex Color" width:180 height:30

    local obj = undefined
    local vp = undefined

    local oldColors = #()
    local oldAlpha = #()
    local hadAlpha = false

    on btnEdit pressed do
    (
        if selection.count != 1 then
        (
            messageBox "Select one object."
        )
        else
        (
            obj = selection[1]

            if classof obj.baseObject != Editable_Mesh do
                convertToMesh obj

            if not (meshop.getMapSupport obj 0) then
            (
                messageBox "Object has no Vertex Color channel."
                obj = undefined
            )
            else
            (
                oldColors = #()
                oldAlpha = #()

                -- Store original vertex colors
                local colorCount = meshop.getNumMapVerts obj 0

                for i = 1 to colorCount do
                    append oldColors (meshop.getMapVert obj 0 i)

                -- Store Vertex Alpha
                hadAlpha = meshop.getMapSupport obj -2

                if hadAlpha do
                (
                    local alphaCount = meshop.getNumMapVerts obj -2

                    for i = 1 to alphaCount do
                        append oldAlpha (meshop.getMapVert obj -2 i)
                )

                -- Make channel 0 completely black
                for i = 1 to colorCount do
                    meshop.setMapVert obj 0 i [0,0,0]

                update obj

                -- Add native Vertex Paint modifier
                vp = VertexPaint()
                vp.mapChannel = 0
                vp.layerIsolated = true

                addModifier obj vp

                max modify mode
                modPanel.setCurrentObject vp

                obj.showVertexColors = true
                obj.vertexColorType = #color

                redrawViews()
            )
        )
    )


    on btnSave pressed do
    (
        if obj == undefined or vp == undefined then
        (
            messageBox "Nothing is being edited."
        )
        else
        (
            local paintedMesh = snapshotAsMesh obj
            local baseMesh = obj.baseObject

            local faceCount = meshop.getNumMapFaces baseMesh 0
            local paintedFaceCount = meshop.getNumMapFaces paintedMesh 0

            if faceCount != paintedFaceCount then
            (
                delete paintedMesh
                messageBox "Mesh topology changed."
            )
            else
            (
                local sums = #()
                local counts = #()

                for i = 1 to oldColors.count do
                (
                    append sums 0.0
                    append counts 0
                )

                -- Read painted grayscale through matching face corners
                for f = 1 to faceCount do
                (
                    local oldFace = meshop.getMapFace baseMesh 0 f
                    local paintedFace = meshop.getMapFace paintedMesh 0 f

                    for corner = 1 to 3 do
                    (
                        local oldIndex = oldFace[corner]
                        local paintedIndex = paintedFace[corner]

                        local c = meshop.getMapVert paintedMesh 0 paintedIndex

                        sums[oldIndex] += c.x
                        counts[oldIndex] += 1
                    )
                )

                delete paintedMesh

                -- Remove temporary Vertex Paint
                deleteModifier obj vp

                -- Restore original R/G and replace only B
                for i = 1 to oldColors.count do
                (
                    local mask = 0.0

                    if counts[i] > 0 do
                        mask = sums[i] / counts[i]

                    local old = oldColors[i]

                    meshop.setMapVert baseMesh 0 i \
                        [old.x, old.y, mask]
                )

                -- Restore Vertex Alpha exactly
                if hadAlpha do
                (
                    for i = 1 to oldAlpha.count do
                        meshop.setMapVert baseMesh -2 i oldAlpha[i]
                )

                update obj

                obj.showVertexColors = true
                obj.vertexColorType = #color

                redrawViews()

                obj = undefined
                vp = undefined
                oldColors = #()
                oldAlpha = #()

                messageBox "Vertex Color B saved."
            )
        )
    )
)

createDialog VertexBPaint 200 100
