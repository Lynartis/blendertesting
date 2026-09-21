//****************
// PROGRAM
//****************
float4 main( const PS_INPUT i ) : SV_TARGET
{
    float MAX_IMPULSE_RADIUS = 350.0f;

    uint   uImpulses         = 0;

    // Blended / additive XY field
    float2 vTotalOffset      = float2( 0.0f, 0.0f );
    float  fTotalWeight      = 0.0f;

    // Radius logic
    float  fCurrentMaxRadius = 0.0f;
    float  fPrevRadius       = 0.0f;


    // --------------------------------------------------------
    // TRAILMAP PIXEL -> WORLD XY
    // --------------------------------------------------------

    float2 uv = i.vTex0;
    uv.y = 1.0f - uv.y;

    float2 vWorld =
        WORLD_PIVOT +
        ((uv - float2( 0.5f, 0.5f )) * UV_TO_WORLD);


    // --------------------------------------------------------
    // CURRENT IMPULSES
    // --------------------------------------------------------

    for( uint j = 0; j < MAX_IMPULSES; ++j )
    {
        if( j < (uint)IMPULSES )
        {
            float2 vDir =
                vWorld -
                GET_IMPULSE_POS( j );

            float fSqrDist =
                1.0f -
                saturate(
                    dot( vDir, vDir ) *
                    GET_IMPULSE_RCP_RADIUS( j )
                );

            if( fSqrDist > 0.0f )
            {
                float fRadius =
                    sqrt(
                        1.0f /
                        GET_IMPULSE_RCP_RADIUS( j )
                    );

                // Smooth additive / weighted vector field
                vTotalOffset +=
                    vDir *
                    fSqrDist;

                fTotalWeight +=
                    fSqrDist;

                // Radius keeps the biggest CURRENT one
                fCurrentMaxRadius =
                    max(
                        fCurrentMaxRadius,
                        fRadius
                    );

                uImpulses++;
            }
        }
    }


    // --------------------------------------------------------
    // PREVIOUS TRAILMAP
    // --------------------------------------------------------

    float4 vPrev =
        texHistory.Sample(
            samplerHistory,
            i.vTex0 - UV_DISPLACEMENT
        );

    float fPrevPower =
        saturate(
            TrailMap_GetPower( vPrev ) -
            DECAY
        );


    // --------------------------------------------------------
    // PREVIOUS HISTORY CONTRIBUTION
    // --------------------------------------------------------

    if( fPrevPower > 0.0f )
    {
        float2 vPrevOffset =
            (vPrev.rg * 2.0f - 1.0f) *
            MAX_IMPULSE_RADIUS;

        fPrevRadius =
            vPrev.b *
            MAX_IMPULSE_RADIUS;

        // History contributes to the blended field
        vTotalOffset +=
            vPrevOffset *
            fPrevPower;

        fTotalWeight +=
            fPrevPower;
    }


    // --------------------------------------------------------
    // OUTPUT
    // --------------------------------------------------------

    float2 vRetOffset =
        float2( 0.5f, 0.5f );

    float fRetRadius = 0.0f;
    float fRetPower  = 0.0f;

    if( fTotalWeight > 0.0f )
    {
        // ----------------------------------------------------
        // RG
        //
        // Blended XY offset field packed into 0..1
        // ----------------------------------------------------

        float2 vOffset =
            vTotalOffset /
            fTotalWeight;

        vRetOffset =
            saturate(
                (vOffset / MAX_IMPULSE_RADIUS) *
                0.5f +
                0.5f
            );


        // ----------------------------------------------------
        // B
        //
        // Maximum radius wins
        // ----------------------------------------------------

        float fMaxRadius =
            max(
                fCurrentMaxRadius,
                fPrevRadius
            );

        fRetRadius =
            saturate(
                fMaxRadius /
                MAX_IMPULSE_RADIUS
            );


        // ----------------------------------------------------
        // A
        //
        // Only reactivate if the CURRENT max radius wins.
        // This prevents a small new impulse from reactivating
        // an older larger sphere.
        // ----------------------------------------------------

        if( uImpulses > 0 )
        {
            fRetPower =
                (fCurrentMaxRadius >= fPrevRadius)
                ? 1.0f
                : fPrevPower;
        }
        else
        {
            fRetPower =
                fPrevPower;
        }
    }


    // --------------------------------------------------------
    // FINAL TRAILMAP
    //
    // RG = blended XY offset field
    // B  = maximum radius
    // A  = power / decay
    // --------------------------------------------------------

    float4 vRet =
        float4(
            0.0f,
            0.0f,
            0.0f,
            0.0f
        );

    vRet.rg = vRetOffset;
    vRet.b  = fRetRadius;
    vRet.a  = fRetPower;

    return vRet;
}
